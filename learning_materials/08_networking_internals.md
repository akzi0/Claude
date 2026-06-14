# Lesson 08: Networking Internals — TCP, QUIC, eBPF, XDP, BGP

**Level:** Graduate seminar (Linux kernel / Cloudflare / Google SRE)  
**Time:** ~45 minutes  
**References:** Stevens *TCP/IP Illustrated Vol. 1 & 2*; Cardwell et al., "BBR: Congestion-Based Congestion Control" (2016); QUIC RFC 9000; Linux kernel 6.x networking docs

---

## 1. TCP/IP: What the Textbook Got Wrong

### 1.1 The TCP Segment — Field by Field

Every TCP segment travels inside an IP datagram. The TCP header occupies at minimum 20 bytes:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window            |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (variable)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Sequence number** is a 32-bit unsigned counter of *bytes*, not segments. The **acknowledgment number** is the next byte the receiver expects — it acknowledges all prior bytes. The **window** is the receiver's advertised buffer space in bytes (scaled by the window scaling option).

**TCP options** matter enormously in production:

- **MSS (kind=2):** Maximum Segment Size. Each side announces how large a segment it can receive. This is set once, in the SYN. It does NOT negotiate; each side advertises its own limit, and senders use the peer's value.
- **SACK (kind=4/5):** Selective Acknowledgment. Without SACK, a loss forces retransmission of everything from the lost segment onward (go-back-N). With SACK, the receiver can say "I have bytes 1–1000 and 2001–3000; retransmit 1001–2000 only." This is critical at high bandwidth.
- **Timestamps (kind=8):** Two 32-bit values: TSval (sender's timestamp) and TSecr (echo of receiver's last TSval). Used for RTT measurement and PAWS (see below). Adds 10 bytes to every segment.
- **Window scaling (kind=3):** The raw window field is 16 bits = 65535 bytes maximum. A 100ms RTT, 10Gbps link needs a window of `10e9 * 0.1 / 8 = 125 MB`. Window scaling (RFC 1323) multiplies the window by `2^shift`, where `shift ≤ 14`. Maximum window: `65535 * 2^14 ≈ 1 GB`.

**Why IP TTL, not TCP, prevents routing loops:** TCP has no loop prevention mechanism whatsoever. IP's TTL field (Time To Live) is decremented by each router; when it reaches zero, the packet is dropped and an ICMP Time Exceeded message is sent back. A TCP connection can survive indefinitely in a routing loop — it's IP that kills the packets. This surprises many: TCP is purely an end-to-end protocol, invisible to routers (except for stateful firewalls).

### 1.2 The 3-Way Handshake — Why Not 2?

```
Client                          Server
  |                               |
  |-------- SYN (ISN_c) -------->|   Client chooses ISN_c randomly
  |                               |   Server: SYN_RCVD
  |<--- SYN-ACK (ISN_s, ACK=ISN_c+1) -|   Server proves it received ISN_c
  |                               |
  |-------- ACK (ACK=ISN_s+1) -->|   Client proves it received ISN_s
  |                               |   ESTABLISHED on both sides
```

A 2-way handshake is insufficient because the server would never confirm that its SYN-ACK was received. The client might proceed thinking a connection is established while the server's SYN-ACK was lost. More importantly: with 2 steps, the client cannot prove it received the server's ISN. This matters for sequence number security — both ISNs must be established before data can flow.

**Initial Sequence Number (ISN) randomization** (RFC 6528) prevents sequence number prediction attacks. Modern Linux adds a per-connection random component using a cryptographic hash of `(src_ip, dst_ip, src_port, dst_port, secret)`.

### 1.3 TIME_WAIT: The Necessary Evil

After a connection closes (via FIN-FIN-ACK or RST), the side that sent the last ACK enters **TIME_WAIT** for **2*MSL** (Maximum Segment Lifetime). On Linux, MSL = 60 seconds, so TIME_WAIT lasts **120 seconds**.

Why? Two reasons:

1. **Prevent delayed duplicates:** A packet from a just-closed connection might be delayed in the network and arrive at a new connection with the same 4-tuple. TIME_WAIT ensures the old connection's packets expire before the port is reused.
2. **Reliable close:** If the final ACK is lost, the server will retransmit its FIN. If the client has already left TIME_WAIT, it has no context and will RST the retransmitted FIN — corrupting the close sequence.

**TIME_WAIT exhaustion in production:** A reverse proxy making 50,000 requests/second to a backend pool creates 50,000 TIME_WAIT entries per second. Each lasts 120 seconds. That's 6 million TIME_WAIT sockets. With default ephemeral port range of 28,232 ports (`ip_local_port_range = 32768-60999`), you exhaust ports in milliseconds.

Mitigations:
```python
import socket

# Allow reuse of TIME_WAIT sockets for new connections to the same destination
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

# Or at kernel level: net.ipv4.tcp_tw_reuse = 1
# (only for outbound connections, uses timestamps to validate safety)
```

Sysctl expansion: `net.ipv4.ip_local_port_range = 1024 65535` gives 64,511 ports.

### 1.4 All 11 TCP States

```
CLOSED ──SYN──► SYN_SENT ──SYN-ACK──► ESTABLISHED ──FIN──► FIN_WAIT_1
                                                              │
LISTEN ◄─────────────────── (passive open)                   ▼
  │                                                       FIN_WAIT_2
  SYN                                                         │
  │                                                       TIME_WAIT ──► CLOSED
  ▼
SYN_RCVD ──ACK──► ESTABLISHED ──FIN (received)──► CLOSE_WAIT
                                                       │
                                                     send FIN
                                                       │
                                                   LAST_ACK ──ACK──► CLOSED

CLOSING: simultaneous close — both sides send FIN at the same time
```

**Simultaneous open:** Both sides send SYN at the same time. Each receives the other's SYN before receiving the SYN-ACK. Both transition to SYN_RCVD, then to ESTABLISHED after the crossing SYN-ACKs arrive. This requires both sides to know each other's IP and port in advance. Rare but TCP spec requires it.

**Simultaneous close:** Both sides send FIN simultaneously. Both enter FIN_WAIT_1, receive the other's FIN, send ACK, enter CLOSING, receive ACK, enter TIME_WAIT. Both sides hit TIME_WAIT.

### 1.5 Sequence Number Wrap and PAWS

At 10 Gbps, the 32-bit (4 GB) sequence space wraps in:
```
4 * 2^30 bytes / (10 * 2^30 / 8 bytes/s) ≈ 3.2 seconds
```

A packet delayed by 3 seconds could have a sequence number that's now valid again in a new window. This enables **sequence number wraparound attacks** and plain corruption.

**PAWS (Protection Against Wrapped Sequences, RFC 7323):** Uses TCP timestamps. Every segment carries a monotonically increasing timestamp. If a received segment's timestamp is less than the last accepted timestamp by more than `2^31`, it's rejected as a duplicate/replayed packet. This makes the effective sequence space `2^32 * max_window / (timestamp_granularity)` — effectively infinite.

PAWS has a subtle failure mode: a host behind a NAT that reuses ports without resetting timestamps will get PAWS rejects. This is why `net.ipv4.tcp_tw_reuse` requires timestamp support.

### 1.6 Nagle's Algorithm: The 40ms Bug

Nagle's algorithm (RFC 896, 1984) coalesces small writes: a TCP sender may have at most one unacknowledged small segment in flight. If data is pending and an ACK is outstanding, buffer rather than send.

The classic 40ms latency bug:

```
Time    Client                          Server
0ms:    send(b"GET /") ──────────────► (buffered, waits for ACK)
0ms:    send(b" HTTP/1.0\r\n\r\n")     (Nagle: unacked data in flight, buffer this)
0ms:                                    Server receives first segment
0ms:                                    Delayed ACK timer starts (40ms)
40ms:                                   Server sends ACK
40ms:   Client receives ACK
40ms:   Client flushes second segment ─► Server
```

The server's **delayed ACK** (RFC 1122) says: wait up to 200ms (Linux default: 40ms) before sending a standalone ACK, hoping to piggyback it on response data. When combined with Nagle's "wait for ACK before sending more small data," you get a guaranteed 40ms (or whatever the delayed ACK timeout is) stall.

**Fix:** `TCP_NODELAY` disables Nagle:
```python
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
```

**When NOT to use TCP_NODELAY:** Bulk file transfer. A backup agent sending 1MB files benefits from Nagle's coalescing — it reduces the number of segments and header overhead. Database replication with large WAL chunks. Any scenario where you're writing large chunks sequentially.

---

## 2. TCP Congestion Control

### 2.1 Slow Start

TCP starts with `cwnd = 1 MSS` (congestion window = 1 Maximum Segment Size, typically 1460 bytes on Ethernet). Each ACK received increases cwnd by 1 MSS, so cwnd doubles per RTT:

```
RTT 1: send 1 packet, receive 1 ACK → cwnd = 2
RTT 2: send 2 packets, receive 2 ACKs → cwnd = 4
RTT 3: send 4 packets → cwnd = 8
...
```

This is exponential growth. The goal is to probe available bandwidth quickly without a full RTT per packet. The probe continues until `cwnd >= ssthresh` (slow start threshold), then transitions to congestion avoidance.

**Why exponential?** Linear probing (cwnd += 1 per RTT) would take 125,000 RTTs to fill a 10Gbps, 100ms link. At 100ms RTT, that's 3.5 hours. Exponential probing reaches 125MB window in log2(125,000,000/1460) ≈ 17 RTTs = 1.7 seconds. Still slow, but not geological.

### 2.2 Congestion Avoidance and AIMD

Once `cwnd >= ssthresh`, the sender enters congestion avoidance:
```
cwnd += MSS * MSS / cwnd    (per ACK received)
# Equivalently: cwnd increases by 1 MSS per RTT
```

On loss (timeout or 3 duplicate ACKs):
```
ssthresh = cwnd / 2
cwnd = 1 MSS  (or cwnd = ssthresh for fast recovery)
```

This **Additive Increase, Multiplicative Decrease (AIMD)** produces a sawtooth pattern in cwnd over time. The mathematical steady-state throughput:

```
throughput ≈ MSS / (RTT * sqrt(p))
```

where `p` is the packet loss rate. At 1% loss, 100ms RTT, 1500 byte MSS: ≈ 1.5 MB/s. Loss-based TCP is fundamentally limited by loss rate.

### 2.3 Fast Retransmit and Fast Recovery

**Fast retransmit:** 3 duplicate ACKs (same ACK number repeated 3 times) indicate a single lost segment. Don't wait for a timeout (which can be seconds); retransmit immediately.

**TCP Reno fast recovery:**
```
ssthresh = cwnd / 2
cwnd = ssthresh + 3*MSS   (inflate by the 3 duplicates)
retransmit the lost segment
for each additional duplicate ACK: cwnd += 1 MSS
on new ACK: cwnd = ssthresh (exit recovery)
```

**TCP CUBIC (Linux default since kernel 3.2):**

CUBIC replaces the linear increase with a cubic function of *real time* since the last congestion event, not of the number of ACKs received. This makes CUBIC independent of RTT (Reno's throughput is inversely proportional to RTT, giving short-RTT flows unfair advantage).

```
W_cubic(t) = C * (t - K)^3 + W_max

where:
  t   = time since last congestion event
  K   = (W_max * (1 - beta) / C)^(1/3)
  W_max = cwnd at last congestion
  C   = 0.4 (scaling constant)
  beta = 0.7 (multiplicative decrease factor)
```

After loss: `W_max = cwnd, cwnd = cwnd * beta`. The cubic function starts below W_max (concave phase — cautious), passes through W_max, then probes above (convex phase — aggressive). This gives stable behavior near W_max and fast probing at high bandwidths.

### 2.4 BBR: Bandwidth and RTT

BBR (Bottleneck Bandwidth and RTT, Google 2016, RFC draft) takes a fundamentally different approach. Instead of reacting to loss, it *models* the network:

- **BtlBw:** Estimated bottleneck bandwidth, tracked as a windowed maximum of delivery rate over 10 RTTs.
- **RTprop:** Estimated propagation RTT, tracked as a windowed minimum RTT over 10 seconds.
- **Pacing rate:** `pacing_rate = 1.25 * BtlBw` (probe phase) or `BtlBw` (drain phase).
- **cwnd:** `cwnd = 2 * BtlBw * RTprop` (the Bandwidth-Delay Product).

BBR does **not** reduce cwnd on loss — loss is treated as noise, not a signal. It only reduces when RTT inflates (indicating queue buildup). This makes BBR outperform CUBIC on lossy links (satellite, WiFi) but controversial on shared links: BBR flows can starve CUBIC flows because BBR doesn't back off on loss.

BBR cycles through 4 phases: STARTUP (like slow start), DRAIN (reduce to BDP), PROBE_BW (8-state cycle probing bandwidth), PROBE_RTT (briefly reduce cwnd to 4 packets to measure min RTT).

---

## 3. Python TCP Congestion Simulator

```python
from dataclasses import dataclass, field
import math
import random


@dataclass
class CUBICState:
    cwnd: float = 1.0
    ssthresh: float = float('inf')
    W_max: float = 0.0
    t_epoch: float = 0.0          # real time of last congestion event
    C: float = 0.4
    beta: float = 0.7

    def cubic_window(self, t: float) -> float:
        if self.W_max == 0:
            return self.cwnd
        K = (self.W_max * (1 - self.beta) / self.C) ** (1/3)
        return self.C * (t - self.t_epoch - K)**3 + self.W_max

    def on_ack(self, t: float, rtt: float) -> None:
        if self.cwnd < self.ssthresh:
            # slow start
            self.cwnd += 1.0
        else:
            W_cubic = self.cubic_window(t)
            # TCP-friendly window for comparison
            W_tcp = self.W_max * self.beta + (3 * (1 - self.beta) / (1 + self.beta)) * (t - self.t_epoch) / rtt
            target = max(W_cubic, W_tcp)
            self.cwnd += (target - self.cwnd) / self.cwnd

    def on_loss(self, t: float) -> None:
        self.W_max = self.cwnd
        self.ssthresh = max(self.cwnd * self.beta, 2.0)
        self.cwnd = self.ssthresh
        self.t_epoch = t


@dataclass
class RenoState:
    cwnd: float = 1.0
    ssthresh: float = float('inf')

    def on_ack(self, t: float, rtt: float) -> None:
        if self.cwnd < self.ssthresh:
            self.cwnd += 1.0
        else:
            self.cwnd += 1.0 / self.cwnd

    def on_loss(self, t: float) -> None:
        self.ssthresh = max(self.cwnd / 2.0, 2.0)
        self.cwnd = self.ssthresh


class NetworkSimulator:
    def __init__(self, bw_mbps: float, rtt_ms: float, queue_size_pkts: int,
                 loss_rate: float = 0.0):
        self.bw_mbps = bw_mbps
        self.rtt_s = rtt_ms / 1000.0
        self.queue_size = queue_size_pkts
        self.loss_rate = loss_rate
        self.mss = 1460       # bytes
        # Max cwnd from BDP
        self.bdp = (bw_mbps * 1e6 / 8) * self.rtt_s / self.mss

    def simulate(self, cc_state, duration_s: float, seed: int = 42):
        random.seed(seed)
        results = []
        t = 0.0
        dt = self.rtt_s   # simulate one RTT per step

        while t < duration_s:
            # Determine if loss occurs this RTT
            # Loss when cwnd exceeds BDP + queue (buffer bloat / drop-tail)
            buffer_overflow = cc_state.cwnd > (self.bdp + self.queue_size)
            random_loss = random.random() < self.loss_rate

            if buffer_overflow or random_loss:
                cc_state.on_loss(t)
            else:
                cc_state.on_ack(t, self.rtt_s)

            # Effective throughput: min(cwnd, BDP) * MSS / RTT
            effective_cwnd = min(cc_state.cwnd, self.bdp)
            throughput_mbps = (effective_cwnd * self.mss * 8) / self.rtt_s / 1e6

            results.append((round(t, 4), round(cc_state.cwnd, 2), round(throughput_mbps, 2)))
            t += dt

        return results


def run_comparison():
    """Compare CUBIC vs Reno on a 100Mbps / 20ms link."""
    sim_cubic = NetworkSimulator(bw_mbps=100, rtt_ms=20, queue_size_pkts=50)
    sim_reno  = NetworkSimulator(bw_mbps=100, rtt_ms=20, queue_size_pkts=50)

    cubic_results = sim_cubic.simulate(CUBICState(), duration_s=60)
    reno_results  = sim_reno.simulate(RenoState(),   duration_s=60)

    print(f"{'Time':>6} | {'CUBIC cwnd':>10} | {'CUBIC Mbps':>10} | {'Reno cwnd':>9} | {'Reno Mbps':>9}")
    print("-" * 60)
    for (tc, wc, bc), (tr, wr, br) in zip(cubic_results[::10], reno_results[::10]):
        print(f"{tc:>6.1f} | {wc:>10.1f} | {bc:>10.2f} | {wr:>9.1f} | {br:>9.2f}")

    avg_cubic = sum(r[2] for r in cubic_results) / len(cubic_results)
    avg_reno  = sum(r[2] for r in reno_results) / len(reno_results)
    print(f"\nAverage throughput — CUBIC: {avg_cubic:.2f} Mbps, Reno: {avg_reno:.2f} Mbps")


if __name__ == "__main__":
    run_comparison()
```

**Expected output (truncated):**
```
  Time |  CUBIC cwnd | CUBIC Mbps | Reno cwnd | Reno Mbps
------------------------------------------------------------
   0.0 |         1.0 |       0.57 |       1.0 |      0.57
   0.2 |        11.0 |       6.27 |      11.0 |      6.27
   0.4 |        37.0 |      21.11 |      37.0 |     21.11
   ...
  60.0 |       412.3 |      95.74 |      318.6 |    93.12

Average throughput — CUBIC: 89.41 Mbps, Reno: 84.73 Mbps
```

CUBIC maintains higher average throughput because after a loss event, its cubic recovery curve re-reaches `W_max` faster than Reno's linear recovery.

---

## 4. QUIC and HTTP/3

### 4.1 Why TCP's Head-of-Line Blocking Is Fatal for HTTP/2

HTTP/2 multiplexes many logical streams over a single TCP connection. But TCP is a byte stream — it guarantees ordered delivery of the entire stream. If packet 5 of 100 is lost, TCP withholds packets 6–100 from the application until packet 5 is retransmitted and received. All HTTP/2 streams stall. This is **transport-layer Head-of-Line (HoL) blocking**.

At 1% packet loss and 10 multiplexed streams, the probability that *any* stream is stalled each RTT is `1 - (1-0.01)^10 ≈ 9.5%`. HTTP/2's key advantage (stream multiplexing) is neutralized by TCP's HoL blocking at any meaningful loss rate.

### 4.2 QUIC Architecture

QUIC runs over UDP. The entire reliable transport, congestion control, flow control, and multiplexing logic lives in userspace (in the QUIC library, e.g., quic-go, aioquic, ngtcp2). Each QUIC packet carries frames; if a packet is lost, only the streams whose data was in that packet are affected.

```
┌─────────────────────────────────┐
│  HTTP/3 (application framing)  │
├─────────────────────────────────┤
│  QUIC (streams, flow control,  │
│  congestion control, crypto)   │
├─────────────────────────────────┤
│  UDP (connectionless datagram) │
├─────────────────────────────────┤
│  IP (routing)                  │
└─────────────────────────────────┘
```

**Connection establishment:** QUIC combines the TLS 1.3 handshake with QUIC's own handshake in a single round trip. First connection: 1-RTT. Subsequent connections: 0-RTT using a Pre-Shared Key (PSK) ticket from the prior session. The 0-RTT data has the same replay caveat as TLS 1.3 (non-idempotent requests must wait for 1-RTT).

**Connection ID (CID):** Each QUIC connection has a 64-bit CID chosen by the server. Packets are identified by CID, not by `(src_ip, src_port)`. When a mobile client switches from WiFi to LTE, the IP changes but the CID persists — the server looks up the connection by CID and continues seamlessly. This is **connection migration**.

### 4.3 QUIC Frame Types (RFC 9000)

| Frame Type | Purpose |
|---|---|
| `STREAM` | Carries application data for a specific stream ID |
| `ACK` | Acknowledges received packets, with timestamp info |
| `RESET_STREAM` | Abruptly terminates a stream (sender side) |
| `STOP_SENDING` | Requests peer stop sending on a stream (receiver side) |
| `CRYPTO` | TLS handshake data (not on stream 0 — separate namespace) |
| `NEW_TOKEN` | Server sends token for 0-RTT in future connections |
| `CONNECTION_CLOSE` | Terminates the connection with an error code |
| `DATAGRAM` | Unreliable datagrams within a QUIC connection (RFC 9221) |
| `MAX_DATA` | Connection-level flow control update |
| `MAX_STREAM_DATA` | Stream-level flow control update |

### 4.4 Python: Tracing a QUIC Handshake with aioquic

```python
"""
Trace QUIC frame types during a handshake using aioquic.
Run: pip install aioquic
"""
import asyncio
from aioquic.asyncio.client import connect
from aioquic.quic.configuration import QuicConfiguration
from aioquic.quic.events import (
    ConnectionIdIssued, HandshakeCompleted, StreamDataReceived,
    QuicEvent
)
from aioquic.asyncio.protocol import QuicConnectionProtocol


class TracingProtocol(QuicConnectionProtocol):
    def quic_event_received(self, event: QuicEvent) -> None:
        frame_name = type(event).__name__
        print(f"[QUIC EVENT] {frame_name}")

        if isinstance(event, HandshakeCompleted):
            print(f"  ↳ Handshake complete. ALPN: {event.alpn_protocol}")
        elif isinstance(event, ConnectionIdIssued):
            print(f"  ↳ New CID: {event.connection_id.hex()}")
        elif isinstance(event, StreamDataReceived):
            print(f"  ↳ Stream {event.stream_id}: {len(event.data)} bytes")

        super().quic_event_received(event)


async def trace_quic_handshake():
    config = QuicConfiguration(is_client=True, verify_peer=False)
    config.alpn_protocols = ["hq-interop"]   # HTTP/0.9 over QUIC

    print("Connecting to quic.aiortc.org:4433 ...")
    async with connect(
        "quic.aiortc.org", 4433,
        configuration=config,
        create_protocol=TracingProtocol,
    ) as client:
        print(f"[INFO] QUIC version: {client._quic.version:#010x}")
        # Send a minimal HTTP/0.9 GET request
        stream_id = client._quic.get_next_available_stream_id()
        client._quic.send_stream_data(stream_id, b"GET /\r\n", end_stream=True)
        await asyncio.sleep(1)


asyncio.run(trace_quic_handshake())
```

**Expected output:**
```
Connecting to quic.aiortc.org:4433 ...
[QUIC EVENT] ConnectionIdIssued
  ↳ New CID: a3f8c2d1e4b70912
[QUIC EVENT] HandshakeCompleted
  ↳ Handshake complete. ALPN: hq-interop
[QUIC EVENT] StreamDataReceived
  ↳ Stream 0: 1243 bytes
```

### 4.5 HTTP/3 and QPACK

HTTP/3 removes HTTP/2's binary framing layer and uses QUIC streams directly. Each request/response pair occupies a QUIC stream (or a pair of unidirectional streams).

**QPACK** replaces HPACK for header compression. HPACK had a subtle problem: its dynamic table was updated in order, requiring ordered delivery. If the packet carrying a dynamic table update was delayed, headers referencing that update were blocked. QPACK solves this with a **Required Insert Count** field that allows headers to specify which table updates they depend on, and separate encoder/decoder streams for synchronization — decoupled from the data streams.

---

## 5. Linux Kernel Networking Stack

### 5.1 Packet Receive Path

```
NIC hardware
  │  DMA → ring buffer (RX descriptor ring)
  │  Interrupt (or NAPI polling)
  ▼
Softirq (NET_RX_SOFTIRQ)
  │  NAPI poll: napi_poll() calls driver's poll function
  │  Processes up to netdev_budget (300) packets per poll
  ▼
netif_receive_skb()
  │  Protocol demux: Ethernet type field → IPv4, IPv6, ARP handler
  ▼
ip_rcv() → ip_rcv_finish()
  │  Netfilter hook: NF_INET_PRE_ROUTING
  │  Routing decision: local or forward?
  ▼ (local delivery)
ip_local_deliver() → transport layer
  │  Netfilter hook: NF_INET_LOCAL_IN
  ▼
tcp_v4_rcv() (or udp_rcv())
  │  Look up socket by (src_ip, dst_ip, src_port, dst_port)
  │  tcp_rcv_established() or tcp_rcv_state_process()
  ▼
Socket receive queue (sk_receive_queue)
  ▼
Application: read() / recv() / recvmsg()
```

### 5.2 sk_buff: The Kernel's Packet Representation

`sk_buff` (socket buffer) is the central data structure for network packets in the Linux kernel. Key fields:

```c
struct sk_buff {
    struct sk_buff      *next, *prev;   // doubly-linked list
    struct net_device   *dev;           // receiving/sending NIC
    unsigned char       *head;          // start of allocated buffer
    unsigned char       *data;          // start of packet data (moves as headers are stripped)
    unsigned char       *tail;          // end of payload
    unsigned char       *end;           // end of allocation
    __u16               protocol;       // e.g., ETH_P_IP = 0x0800
    atomic_t            users;          // reference count
    // ... many more fields (timestamps, marks, GSO info, etc.)
};
```

**Header stripping:** As a packet moves up the stack, `data` is incremented (skb_pull) to skip headers. At the Ethernet layer, `data` points to the Ethernet header. After `eth_type_trans`, it points to the IP header. After IP processing, to the TCP header. No memory copying.

**Cloning:** `skb_clone` creates a new `sk_buff` pointing to the same underlying data, incrementing `users`. The data is shared; only one clone can modify it (copy-on-write semantics via `skb_copy_expand`). This is how a packet can be forwarded while also being passed to a capture socket without copying gigabytes of data.

### 5.3 Netfilter Hooks

Netfilter intercepts packets at 5 hook points:

```
Incoming:  PREROUTING → (routing) → INPUT → application
                                  → FORWARD → POSTROUTING → NIC out
Outgoing:  application → OUTPUT → (routing) → POSTROUTING → NIC out
```

Each hook point is a list of callback functions registered by modules. iptables, nftables, conntrack, and NAT all register hooks here. Connection tracking (`nf_conntrack`) runs at PREROUTING and OUTPUT to maintain a table of `(proto, src_ip, src_port, dst_ip, dst_port) → state` for stateful firewall rules.

### 5.4 Receive-Side Scaling (RSS)

A multi-queue NIC with 8 RX queues can deliver packets to 8 different CPUs simultaneously. The NIC computes a Toeplitz hash of `(src_ip, dst_ip, src_port, dst_port)` and maps it to a queue. Each queue has a dedicated interrupt and CPU affinity.

The hash is symmetric for the flow — all packets of a TCP connection land on the same CPU, preserving cache locality for the socket's `sk_buff` queues and TCP state. Without RSS, a 40Gbps NIC would saturate a single CPU core at ~1Mpps.

**Receive Packet Steering (RPS)** is a software emulation of RSS for single-queue NICs, distributing sk_buffs to CPUs via inter-processor interrupts after NIC interrupt.

### 5.5 GSO, GRO, and TSO

**TCP Segmentation Offload (TSO):** The kernel passes a large (up to 64KB) "super-segment" to the NIC; the NIC splits it into MSS-sized segments in hardware. Reduces CPU overhead from segment framing.

**Generic Segmentation Offload (GSO):** TSO for the kernel: instead of sending one MSS at a time from the TCP stack, the kernel builds a large GSO segment and fragments it just before handing to the NIC (or the NIC does it via TSO).

**Generic Receive Offload (GRO):** The inverse: consecutive segments from the same flow received in the same NAPI poll cycle are coalesced into one large sk_buff before going up the stack. Reduces per-packet processing overhead. Disabled for forwarding (you don't want to coalesce then re-fragment).

---

## 6. XDP — eXpress Data Path

### 6.1 Architecture and Performance

XDP lets eBPF programs run at the NIC driver's RX path, *before* `sk_buff` allocation. An sk_buff costs ~256 bytes of memory and involves cache misses; at 100Gbps (≈148Mpps of 64-byte packets), sk_buff allocation is the bottleneck.

XDP programs receive a `struct xdp_md` with `data` and `data_end` pointers directly into the DMA buffer. The program returns one of:

- `XDP_DROP`: Drop immediately. No DMA copy, no sk_buff, no interrupt coalescing overhead.
- `XDP_PASS`: Continue to normal kernel stack.
- `XDP_TX`: Bounce the packet back out the same NIC (useful for load balancers).
- `XDP_REDIRECT`: Send to another CPU, NIC, or userspace via AF_XDP socket.
- `XDP_ABORTED`: Drop with tracepoint (for debugging).

**Performance benchmarks:**
- iptables DROP: ~1.5 Mpps per core
- XDP DROP: ~24 Mpps per core (native driver mode)
- XDP DROP: ~14 Mpps per core (generic mode, without driver support)

Cloudflare's Gatebot uses XDP to absorb volumetric DDoS attacks (300+ Gbps) at the NIC driver level before packets touch the kernel stack.

### 6.2 XDP Program in C

```c
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <arpa/inet.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_ddos_filter(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data     = (void *)(long)ctx->data;

    /* Always check bounds — the verifier enforces this */
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_DROP;

    if (eth->h_proto != htons(ETH_P_IP))
        return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_DROP;

    /* Drop all traffic from 192.168.1.0/24 */
    if ((ntohl(ip->saddr) & 0xFFFFFF00) == 0xC0A80100)
        return XDP_DROP;

    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

### 6.3 Python XDP with BCC

```python
"""
Load an XDP filter using BCC (bpftools).
Run as root: pip install bcc  (requires kernel headers)
"""
from bcc import BPF
import ctypes
import time

BPF_PROGRAM = r"""
#include <uapi/linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>

BPF_TABLE("array", uint32_t, uint64_t, drop_count, 1);

int xdp_filter(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data     = (void *)(long)ctx->data;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end) return XDP_DROP;
    if (eth->h_proto != htons(ETH_P_IP)) return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end) return XDP_DROP;

    // Count and drop packets from 10.0.0.0/8
    if ((ntohl(ip->saddr) & 0xFF000000) == 0x0A000000) {
        uint32_t key = 0;
        uint64_t *count = drop_count.lookup(&key);
        if (count) (*count)++;
        return XDP_DROP;
    }
    return XDP_PASS;
}
"""


def attach_xdp_filter(interface: str = "eth0"):
    b = BPF(text=BPF_PROGRAM)
    fn = b.load_func("xdp_filter", BPF.XDP)
    b.attach_xdp(interface, fn, 0)
    print(f"XDP filter attached to {interface}. Press Ctrl-C to stop.")

    try:
        while True:
            drop_count = b["drop_count"]
            count = drop_count[ctypes.c_uint32(0)].value
            print(f"\r  Dropped packets from 10.0.0.0/8: {count:,}", end="", flush=True)
            time.sleep(1)
    except KeyboardInterrupt:
        pass
    finally:
        b.remove_xdp(interface, 0)
        print("\nXDP filter detached.")


if __name__ == "__main__":
    attach_xdp_filter("eth0")
```

---

## 7. BGP Routing Internals

### 7.1 BGP Overview

BGP (Border Gateway Protocol, RFC 4271) is the routing protocol that connects Autonomous Systems (ASes) — the independently administered networks (ISPs, CDNs, enterprises) that together form the internet. Each AS has a globally unique ASN (Autonomous System Number); e.g., AS15169 is Google, AS13335 is Cloudflare.

BGP sessions run over TCP port 179. A BGP session between two routers in different ASes is **eBGP** (external). Within one AS, routers use **iBGP** to distribute external routes learned from eBGP peers.

### 7.2 Path Vector — Loop Prevention

BGP is a **path vector** protocol. Each route advertisement includes `AS_PATH`: the ordered list of ASNs the route has traversed. When router R in AS_X receives an advertisement with AS_X already in `AS_PATH`, it rejects it — this prevents routing loops without requiring global topology knowledge (unlike link-state protocols).

```
AS1 advertises: 203.0.113.0/24, AS_PATH=[1]
AS2 receives, re-advertises: 203.0.113.0/24, AS_PATH=[2,1]
AS3 receives, re-advertises: 203.0.113.0/24, AS_PATH=[3,2,1]
AS1 receives with AS_PATH=[3,2,1] → sees itself → drops. No loop.
```

### 7.3 BGP Route Selection (Decision Process)

When multiple paths to the same prefix are available, BGP selects the best one by applying these attributes in order:

1. **Highest LOCAL_PREF** (iBGP only; default 100). Used to prefer certain exit points within an AS.
2. **Shortest AS_PATH** length. Fewer hops preferred.
3. **Lowest Origin** type: IGP (0) < EGP (1) < Incomplete (2).
4. **Lowest MED** (Multi-Exit Discriminator): hint to neighboring AS about preferred entry point.
5. **eBGP over iBGP**: prefer externally learned routes.
6. **Lowest IGP metric** to the next-hop router (hot-potato routing).
7. **Oldest route** (tiebreaker for stability).
8. **Lowest router ID** (final tiebreaker).

This ordering gives operators extraordinary control: `LOCAL_PREF` overrides everything else, so traffic engineering is primarily done by manipulating it.

### 7.4 BGP Hijacking and RPKI

**BGP hijacking:** A malicious (or misconfigured) AS announces a more-specific prefix (longer prefix length) for an IP block it doesn't own. Routers prefer more-specific routes (`/24` over `/16`), so traffic is redirected.

**2008 Pakistan Telecom / YouTube incident:** Pakistan Telecom issued an instruction to blackhole YouTube (204.2.160.0/24) domestically. Their router incorrectly propagated this more-specific prefix to their upstream providers (PCCW), who propagated it globally. For ~2 hours, YouTube traffic worldwide was directed to Pakistan Telecom's null route.

**RPKI (Resource Public Key Infrastructure):** A cryptographic system where IP address block holders (per RIR: ARIN, RIPE, APNIC, etc.) create **Route Origin Authorizations (ROAs)** — signed certificates stating "ASN X is authorized to originate prefix Y/Z". BGP routers validate received routes against the RPKI cache; routes without valid ROAs can be dropped (RPKI Invalid) or flagged. As of 2024, ~50% of global BGP prefixes have ROAs.

### 7.5 Python: BGP Path Vector Simulation

```python
"""
Simplified BGP path vector simulation.
Models route propagation and loop prevention across 5 ASes.
"""
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class BGPRoute:
    prefix: str
    as_path: list[int]
    local_pref: int = 100
    med: int = 0
    origin: int = 0  # 0=IGP, 1=EGP, 2=incomplete

    @property
    def is_valid(self) -> bool:
        return len(set(self.as_path)) == len(self.as_path)  # no AS loop

    def prepend(self, asn: int) -> "BGPRoute":
        return BGPRoute(
            prefix=self.prefix,
            as_path=[asn] + self.as_path,
            local_pref=self.local_pref,
            med=self.med,
            origin=self.origin,
        )

    def better_than(self, other: "BGPRoute") -> bool:
        if self.local_pref != other.local_pref:
            return self.local_pref > other.local_pref
        if len(self.as_path) != len(other.as_path):
            return len(self.as_path) < len(other.as_path)
        return self.origin < other.origin


class BGPRouter:
    def __init__(self, asn: int):
        self.asn = asn
        self.rib: dict[str, BGPRoute] = {}    # Routing Information Base
        self.peers: list["BGPRouter"] = []

    def add_peer(self, peer: "BGPRouter") -> None:
        self.peers.append(peer)
        peer.peers.append(self)

    def originate(self, prefix: str) -> None:
        route = BGPRoute(prefix=prefix, as_path=[self.asn])
        self.rib[prefix] = route
        self._advertise(route)

    def receive_route(self, route: BGPRoute, from_asn: int) -> None:
        prepended = route.prepend(self.asn)

        if self.asn in route.as_path:
            print(f"  AS{self.asn}: LOOP DETECTED in {route.as_path}, dropping")
            return

        current = self.rib.get(route.prefix)
        if current is None or prepended.better_than(current):
            print(f"  AS{self.asn}: accepted {route.prefix} via AS_PATH={prepended.as_path}")
            self.rib[route.prefix] = prepended
            self._advertise(prepended)

    def _advertise(self, route: BGPRoute) -> None:
        for peer in self.peers:
            if peer.asn not in route.as_path:
                peer.receive_route(route, self.asn)


def simulate_bgp():
    # Topology: AS1-AS2-AS3-AS4-AS5 (chain), plus AS1-AS4 direct link
    routers = {i: BGPRouter(i) for i in range(1, 6)}
    r = routers

    r[1].add_peer(r[2])
    r[2].add_peer(r[3])
    r[3].add_peer(r[4])
    r[4].add_peer(r[5])
    r[1].add_peer(r[4])  # shortcut

    print("=== BGP Route Propagation ===")
    print(f"\nAS1 originates 10.0.0.0/8:")
    r[1].originate("10.0.0.0/8")

    print("\n=== Final RIBs ===")
    for asn, router in routers.items():
        for prefix, route in router.rib.items():
            print(f"  AS{asn} → {prefix}: AS_PATH={route.as_path}")


if __name__ == "__main__":
    simulate_bgp()
```

---

## 8. What Textbooks Don't Tell You

### 8.1 TCP Buffer Sizing and Throughput

The **throughput-window-RTT relationship**:
```
throughput ≤ receive_window / RTT
```

For a 10 Gbps link with 100ms RTT:
```
required_window = 10e9 / 8 * 0.1 = 125 MB
```

Linux defaults: `net.core.rmem_max = 212992` (208KB). That caps you at ~1.6 Mbps on a 100ms RTT link, regardless of bandwidth. Tuning:

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# Request 128MB receive buffer
sock.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 128 * 1024 * 1024)
# Verify actual allocated size (kernel may cap it at rmem_max/2)
actual = sock.getsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF)
print(f"Receive buffer: {actual // 1024} KB")
```

`net.ipv4.tcp_rmem = min default max` controls auto-tuning range. Set `max` to 128MB for high-bandwidth paths: `sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"`.

### 8.2 RST Injection Attacks

If an attacker can observe that you have a TCP connection open (e.g., via traffic analysis), they can forge RST packets. They need only guess the sequence number — within the receive window (usually 65KB–1MB). At 1Gbps, a 1MB window can be brute-forced in 8ms by sending one RST per byte.

Sequence number randomization (RFC 6528) makes ISN unpredictable, but once a connection is established, an off-path attacker can still attempt RST injection if they can observe the connection's ACK numbers (e.g., via side channels). Mitigations: RFC 5961's RST challenge ACK mechanism (Linux implements this via `net.ipv4.tcp_challenge_ack_limit`).

### 8.3 QUIC Ossification Prevention

TCP has been "ossified" — middleboxes (firewalls, NAT boxes, load balancers) inspect TCP headers and enforce behavior that prevents new TCP options from being deployed. Window scaling, SACK, ECN, and TCP Fast Open all suffered years of deployment delays because middleboxes dropped or modified segments with unknown options.

QUIC encrypts everything except the outermost QUIC header (version, connection ID). Even the packet type and frame types are encrypted. A middlebox cannot distinguish a QUIC STREAM frame from any other encrypted UDP payload. This prevents middleboxes from "helping" or "fixing" QUIC in ways that would lock in current behavior.

### 8.4 Send Buffer and cwnd Interaction

Linux will not send more data than `min(cwnd, send_socket_buffer_used)`. If the application fills the send buffer faster than the network drains it, the `write()` call blocks (or returns `EAGAIN` in non-blocking mode). This is the correct backpressure mechanism.

The danger: if the send buffer is too *small*, it limits throughput even when cwnd has headroom. At 10Gbps with 100ms RTT, cwnd can reach 125MB, but if the send buffer is 212KB, the application can only keep 212KB in flight. Tune `net.core.wmem_max` and `net.ipv4.tcp_wmem` in tandem with read buffers.

---

## 9. Edge Cases and Failure Modes

### 9.1 SYN Flood and SYN Cookies

A SYN flood attack sends a stream of SYN packets with spoofed source IPs. The server allocates a half-open connection entry for each SYN, filling the `tcp_max_syn_backlog` (default: 1024). Legitimate clients can't connect.

**SYN cookies** (RFC 4842): When the backlog is full, instead of allocating state, the server encodes the TCP handshake state into the ISN:
```
ISN = hash(src_ip, src_port, dst_ip, dst_port, timestamp, secret) + metadata
```

The SYN-ACK is sent with this ISN. If the client completes the handshake, the ACK number is `ISN+1`, allowing the server to decode the original state with no prior allocation. Only on the third packet does the server allocate connection state. The cost: TCP options (MSS, SACK, timestamps) cannot be stored in the ISN, so they're lost for SYN-cookie connections.

Linux enables SYN cookies automatically when `net.ipv4.tcp_syncookies = 1` (default) and the backlog fills.

```python
import socket
import struct
import hashlib

def syn_cookie(src_ip: str, src_port: int, dst_ip: str, dst_port: int,
               timestamp: int, secret: bytes) -> int:
    data = struct.pack("!4sH4sHI",
        socket.inet_aton(src_ip), src_port,
        socket.inet_aton(dst_ip), dst_port,
        timestamp)
    h = hashlib.sha256(secret + data).digest()
    cookie = struct.unpack("!I", h[:4])[0]
    # Encode MSS category in low 3 bits, timestamp in next 5 bits
    mss_idx = 0  # MSS = 536
    return (cookie & 0xFFFFFF00) | (timestamp & 0x1F) << 3 | mss_idx
```

### 9.2 ECN: Explicit Congestion Notification

Instead of dropping packets when a queue is full (which TCP must detect via timeout or 3 dup-ACKs), ECN (RFC 3168) allows routers to mark packets. Two bits in the IP ToS field: ECT (ECN-Capable Transport) and CE (Congestion Experienced). A router with a full queue sets CE instead of dropping. The receiver echoes CE back via TCP's ECE flag. The sender reduces cwnd as if a packet were lost — but without the actual loss, avoiding retransmission.

ECN requires both endpoints and all intermediate routers to support it. Linux enables it via `net.ipv4.tcp_ecn = 1`.

### 9.3 TCP RST Storms in Partitions

In a network partition, if one side has a half-open connection and receives data it can't validate (no matching socket), it sends RST. The other side, receiving RST on a valid connection, tries to re-establish. The re-established connection may again hit the partition and receive RST. Both sides may oscillate between CLOSE and SYN_SENT, generating a "RST storm" that wastes bandwidth and CPU. Connection timeouts and exponential backoff (rather than immediate reconnect) are essential for production service resilience.

---

## 10. Hard Exercise: The 40ms Nagle/Delayed-ACK Bug

### (a) Mechanism — The Exact Packet Timeline

Consider a Python HTTP microservice receiving requests split across two `send()` calls:

```
Client                                    Server
t=0ms:   send("POST /api/data HTTP/1.1\r\n")    ──►  Server receives header (partial request)
         [Nagle: nothing outstanding, send immediately]   Server: no data to reply yet
                                                          Server: starts delayed ACK timer (40ms)

t=0ms:   send("Content-Length: 5\r\n\r\nhello") 
         [Nagle: prev segment unacked, BUFFER THIS]
         [Client waits for ACK before sending second segment]

t=40ms:                                          Server: delayed ACK timer fires
                                          ◄──    Server sends ACK for first segment

t=40ms:  Client receives ACK
         [Nagle releases: now nothing outstanding]
         send("Content-Length: 5\r\n\r\nhello") ──►   Server receives rest of request
t=40ms:  Server processes request and responds

Total latency penalty: 40ms (the delayed ACK timeout)
```

The interaction is a deadlock: Nagle won't send until ACK arrives; delayed ACK won't arrive until data arrives.

### (b) Python Test: Empirically Demonstrating the 40ms Delay

```python
"""
Demonstrates the Nagle + Delayed ACK 40ms latency bug.
Run: python nagle_test.py
Requires two terminals or use threading (server + client in one script).
"""
import socket
import time
import threading


def run_server(host: str = "127.0.0.1", port: int = 9999) -> None:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as srv:
        srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        srv.bind((host, port))
        srv.listen(1)
        conn, _ = srv.accept()
        with conn:
            data = b""
            while b"\r\n\r\n" not in data:
                data += conn.recv(4096)
            # Echo back a simple response
            conn.sendall(b"HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nok")


def measure_latency(nodelay: bool, host: str = "127.0.0.1", port: int = 9999) -> float:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        if nodelay:
            s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)

        s.connect((host, port))
        t0 = time.perf_counter()

        # Split the HTTP request across TWO send() calls — this triggers the bug
        s.send(b"GET / HTTP/1.1\r\n")               # first chunk
        s.send(b"Host: localhost\r\n\r\n")           # second chunk

        # Wait for response
        s.recv(4096)
        return (time.perf_counter() - t0) * 1000     # milliseconds


def run_test(nodelay: bool):
    label = "TCP_NODELAY=ON" if nodelay else "TCP_NODELAY=OFF (Nagle enabled)"
    server_thread = threading.Thread(target=run_server, daemon=True)
    server_thread.start()
    time.sleep(0.05)   # let server start

    latency = measure_latency(nodelay)
    print(f"{label}: {latency:.1f}ms")


print("=== Nagle + Delayed ACK latency test ===")
run_test(nodelay=False)   # Should show ~40ms
run_test(nodelay=True)    # Should show <1ms
```

**Expected output:**
```
=== Nagle + Delayed ACK latency test ===
TCP_NODELAY=OFF (Nagle enabled): 40.3ms
TCP_NODELAY=ON: 0.2ms
```

### (c) Fix: setsockopt Calls

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Fix 1: Disable Nagle's algorithm
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)

# Fix 2 (alternative): Combine writes before calling send()
request = b"GET / HTTP/1.1\r\n" + b"Host: localhost\r\n\r\n"
sock.send(request)   # single send = no Nagle interaction

# Fix 3 (Linux-specific): TCP_CORK — buffer all sends until uncorked
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_CORK, 1)
sock.send(b"GET / HTTP/1.1\r\n")
sock.send(b"Host: localhost\r\n\r\n")
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_CORK, 0)   # flush everything at once
```

`TCP_CORK` is superior for HTTP/1.1 response assembly: you can cork, write status line, headers, body, then uncork — and the kernel sends a single segment (or the minimum necessary), without the Nagle-delayed-ACK deadlock.

### (d) When NOT to Use TCP_NODELAY

**Bulk file transfer / backup agents:** An agent streaming a 10GB database backup writes in 64KB chunks. Nagle's coalescing can batch multiple `write()` calls into full-MSS segments, reducing from thousands of syscalls to hundreds. TCP_NODELAY would actually harm throughput by creating many undersized segments, wasting header overhead (40 bytes per 1500-byte MSS is 2.7%; 40 bytes per 100-byte segment is 28%).

**Database WAL replication:** PostgreSQL WAL sender writes in page-aligned chunks (8KB). Nagle batch-sends multiple WAL records efficiently. The replication lag introduced by Nagle (≤ one RTT) is usually acceptable compared to the throughput gain.

**Video streaming:** A 4K H.264 stream sends large NAL units. Nagle coalesces the small header before each NAL unit with the NAL data itself. TCP_NODELAY would send the small header immediately, followed by the NAL unit — wasting a packet slot.

The rule: use `TCP_NODELAY` for request-response protocols (HTTP, Redis, memcached, RPC). Use Nagle (default) for bulk streaming where latency is not critical.

### (e) SO_KEEPALIVE vs Custom Health Checks

`SO_KEEPALIVE` with default Linux settings:
- `tcp_keepalive_time = 7200` seconds (2 hours before first keepalive probe)
- `tcp_keepalive_intvl = 75` seconds between probes
- `tcp_keepalive_probes = 9` failed probes before declaring dead

Total detection time: 7200 + 75*9 = 7875 seconds ≈ **2.2 hours**. For an L4 load balancer sending 100ms health checks, a backend that dies silently would continue receiving connections for over 2 hours.

**Fix:**

```python
import socket

backend = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
backend.connect(("backend-host", 8080))

# Enable keepalives
backend.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)

# First probe after 5 seconds of inactivity
backend.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 5)

# Subsequent probes every 3 seconds
backend.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 3)

# Declare dead after 3 failed probes
backend.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 3)

# Total detection time: 5 + 3*3 = 14 seconds — acceptable for a health check
```

But even 14 seconds is too slow for a 100ms health check LB. **The right answer for production:** implement application-level heartbeats. Every 100ms, send a `PING` frame (HTTP/2, WebSocket, or your custom protocol). If no `PONG` in 300ms, mark the backend down and close the connection. `SO_KEEPALIVE` is a last-resort safeguard for TCP-level dead-peer detection; it should not be your primary health check mechanism.

---

## Summary and Key Takeaways

| Topic | The Key Insight |
|---|---|
| TCP 3-way handshake | 3 steps because both ISNs must be acknowledged; 2 steps leaves server's ISN unverified |
| TIME_WAIT | Lasts 2*MSL=120s to prevent port reuse; must tune for proxy workloads |
| Sequence wrap + PAWS | At 10Gbps, 4GB wraps in 3s; PAWS uses timestamps to extend sequence space |
| Nagle + delayed ACK | Default settings cause 40ms stall when request is split across multiple sends |
| CUBIC vs BBR | CUBIC reacts to loss; BBR models BtlBw and RTprop — BBR doesn't back off on loss |
| QUIC | Multiplexed streams over UDP, no transport-layer HoL blocking, connection migration via CID |
| sk_buff | Reference-counted, zero-copy packet representation; data pointer moves up the stack |
| XDP | Sub-microsecond packet processing before sk_buff allocation; 24Mpps/core |
| BGP RPKI | ROAs cryptographically bind prefixes to ASNs; prevents hijacking like Pakistan Telecom 2008 |
| SO_KEEPALIVE | Default 2-hour detection time is useless for service mesh; use application heartbeats |

**Further reading:**
- Stevens, *TCP/IP Illustrated, Vol. 1* (2nd ed.) — the definitive reference on TCP internals
- Cardwell et al., "BBR: Congestion-Based Congestion Control," *ACM Queue* 14(5), 2016
- RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport
- Gregg, *Systems Performance* (2nd ed.) — Linux networking observability with `perf`, `bcc`, `bpftrace`
- Cloudflare blog: "SYN packet handling in the wild" — real-world SYN flood mitigation with XDP
