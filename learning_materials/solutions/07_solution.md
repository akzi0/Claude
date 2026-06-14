# Solution: Lesson 07 — Cryptography Internals
## Hard Exercises: ECDSA Nonce Reuse, Schnorr ZKP, and TLS 1.3 Forward Secrecy

---

## 1. Full Exercise Restatement

**Exercise (a) — ECDSA Nonce Reuse Key Recovery:**
Alice signs two different messages `m1` and `m2` using the same nonce `k` in ECDSA with secp256k1. Given the two signatures `(r, s1)` and `(r, s2)` and the message hashes `e1 = H(m1)`, `e2 = H(m2)`, recover Alice's private key `d`.

**Exercise (b) — Schnorr Proof Test Suite:**
Implement and verify a complete test suite for the Schnorr zero-knowledge proof system:
1. Generate random secret; create proof; verify proof succeeds
2. Tamper with `s` → verify proof fails
3. Additional edge cases: wrong commitment A, wrong public key Y, edge secrets

**Exercise (c) — TLS 1.3 Forward Secrecy Analysis:**
If an attacker steals the TLS 1.3 server's certificate private key, can they decrypt recorded past sessions? Explain using the precise TLS 1.3 key schedule.

---

## 2. Conceptual Solution Walkthrough

### Exercise (a): ECDSA Nonce Reuse — Mathematical Derivation

ECDSA signature generation for message hash `e = H(m)` with private key `d`, nonce `k`:

```
R = k·G          # point multiplication
r = R.x mod n   # x-coordinate of R, reduced mod group order n
s = k⁻¹(e + r·d) mod n
```

Given two signatures with the **same nonce k** (same `r`, different `s1`, `s2`):

```
s1 = k⁻¹(e1 + r·d) mod n   ... (1)
s2 = k⁻¹(e2 + r·d) mod n   ... (2)
```

Subtract (2) from (1):

```
s1 - s2 = k⁻¹(e1 - e2) mod n
k(s1 - s2) = e1 - e2 mod n
k = (e1 - e2) · (s1 - s2)⁻¹ mod n
```

Once `k` is known, recover `d` from equation (1):

```
s1·k = e1 + r·d mod n
r·d = s1·k - e1 mod n
d = (s1·k - e1) · r⁻¹ mod n
```

**The attack:** Two equations, two unknowns (`k`, `d`). Nonce reuse reduces to a system of linear equations over Z_n, solvable in O(1) modular arithmetic operations. This is why nonce reuse is catastrophic — it reduces a hard ECDLP problem to trivial linear algebra.

**Historical impact:** Sony PS3 used a broken ECDSA implementation that generated the same `k` for every signature (derived from a constant, not a random source). This allowed reverse engineers (fail0verflow) to extract the root key that authenticated all PS3 software, enabling unsigned code execution. The 2016 research on weak Bitcoin keys found thousands of reused nonces in early transactions, enabling recovery of private keys from publicly visible blockchain transactions.

### Exercise (b): Schnorr Zero-Knowledge Proof

The Schnorr identification protocol allows a prover to convince a verifier they know a secret `x` such that `Y = x·G`, without revealing `x`.

**Protocol (Fiat-Shamir non-interactive version):**

1. Prover picks random nonce `r ← Z_n`
2. Computes commitment `A = r·G`
3. Computes challenge `c = H(G || Y || A)` (Fiat-Shamir: the "random oracle" is a hash)
4. Computes response `s = r + c·x mod n`
5. Proof is `(Y, A, s)`

**Verification:**
- Recompute `c = H(G || Y || A)`
- Check: `s·G == A + c·Y`

Why this works:
```
s·G = (r + c·x)·G = r·G + c·x·G = A + c·Y  ✓
```

**Soundness:** If a cheating prover can produce valid `(A, s)` for the same `A` but two different challenges `c` and `c'` (the "rewinding argument"), they can solve for `x`:
```
s  = r + c·x    s' = r + c'·x
s - s' = (c - c')·x
x = (s - s') · (c - c')⁻¹ mod n
```

Since computing `s` for a given `c` without knowing `x` requires solving ECDLP, a cheating prover cannot succeed except with negligible probability.

**Zero-knowledge:** The proof `(Y, A, s)` is simulable without `x`: pick random `s`, pick random `c`, set `A = s·G - c·Y`. This simulation is indistinguishable from real proofs, so the proof reveals nothing about `x`.

### Exercise (c): TLS 1.3 Forward Secrecy Analysis

The TLS 1.3 key schedule (RFC 8446 §7.1):

```
early_secret = HKDF-Extract(0, PSK_or_0)
     │
     ▼ Derive-Secret(early_secret, "derived", "")
     │
handshake_secret = HKDF-Extract(derived_secret, shared_secret)
     │                                          ↑
     │                                     ECDH(client_ephemeral, server_ephemeral)
     ▼ Derive-Secret(handshake_secret, "c hs traffic", transcript_hash)
     │ → client_handshake_traffic_secret
     │   → client_handshake_key, client_handshake_iv
     │
     ▼ Derive-Secret(handshake_secret, "derived", "")
master_secret = HKDF-Extract(derived_secret, 0)
     │
     ▼ Derive-Secret(master_secret, "c ap traffic", transcript_hash)
       → client_application_traffic_secret_0
         → client_write_key, client_write_iv  (encrypts all application data)
```

**What the certificate private key is used for:**
```python
# In the server's Certificate + CertificateVerify message:
cert_verify_data = b"TLS 1.3, server CertificateVerify" + b"\x00" + transcript_hash
signature = ECDSA_Sign(cert_private_key, SHA256(cert_verify_data))
```

The certificate private key signs the transcript hash. It is **not** mixed into any key derivation.

**Critical observation:** `shared_secret = ECDH(client_ephemeral_private, server_ephemeral_public) = ECDH(server_ephemeral_private, client_ephemeral_public)`.

The `client_ephemeral_private` and `server_ephemeral_private` are:
1. Generated fresh for each connection
2. Used only to compute `shared_secret`  
3. **Securely deleted immediately after** `shared_secret` is computed

**With stolen certificate private key, attacker can:**
- Forge `CertificateVerify` signatures → impersonate the server to future clients
- Perform active MITM on new connections
- Break future sessions by impersonating server identity

**With stolen certificate private key, attacker cannot:**
- Compute `shared_secret` from recorded traffic (would require ephemeral private key)
- Derive `handshake_secret` or `master_secret` (derive from `shared_secret`)
- Decrypt recorded past sessions (no access to the ephemeral keys)

**The 0-RTT exception:** 0-RTT data is encrypted under a key derived from a PSK (session resumption ticket). If the PSK is compromised (server's session ticket encryption key is stolen), 0-RTT data from past sessions can be decrypted. Moreover, 0-RTT data has no anti-replay protection — an attacker can replay it to the server. This is why high-security applications disable 0-RTT.

---

## 3. Full Python Implementation

```python
"""
Cryptography exercises: ECDSA nonce reuse attack, Schnorr ZKP,
TLS 1.3 forward secrecy analysis.

Uses only Python standard library (no external crypto libraries).
For educational purposes ONLY — do not use these implementations in production.
"""

from __future__ import annotations
import hashlib
import secrets
import hmac
import struct
from dataclasses import dataclass
from typing import Optional


# ── Elliptic curve: secp256k1 ──────────────────────────────────────────────────

# secp256k1 parameters (Bitcoin, Ethereum, PS3)
_p = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F
_a = 0
_b = 7
_n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
_Gx = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798
_Gy = 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8


@dataclass(frozen=True)
class Point:
    """A point on the secp256k1 curve, or the point at infinity."""
    x: Optional[int]
    y: Optional[int]

    @property
    def is_infinity(self) -> bool:
        return self.x is None and self.y is None

    def __add__(self, other: 'Point') -> 'Point':
        if self.is_infinity: return other
        if other.is_infinity: return self
        if self.x == other.x and self.y != other.y:
            return Point(None, None)   # additive inverse: point at infinity
        if self.x == other.x:         # self == other: point doubling
            return _point_double(self)
        return _point_add(self, other)

    def __rmul__(self, scalar: int) -> 'Point':
        return _scalar_mul(scalar, self)

    def __mul__(self, scalar: int) -> 'Point':
        return _scalar_mul(scalar, self)


O = Point(None, None)   # point at infinity (identity element)
G = Point(_Gx, _Gy)    # secp256k1 generator point


def _modinv(a: int, m: int) -> int:
    return pow(a, -1, m)


def _point_add(P: Point, Q: Point) -> Point:
    """Add two distinct points on secp256k1."""
    lam = (Q.y - P.y) * _modinv(Q.x - P.x, _p) % _p
    x3 = (lam * lam - P.x - Q.x) % _p
    y3 = (lam * (P.x - x3) - P.y) % _p
    return Point(x3, y3)


def _point_double(P: Point) -> Point:
    """Double a point on secp256k1 (tangent formula; a=0)."""
    lam = (3 * P.x * P.x) * _modinv(2 * P.y, _p) % _p
    x3 = (lam * lam - 2 * P.x) % _p
    y3 = (lam * (P.x - x3) - P.y) % _p
    return Point(x3, y3)


def _scalar_mul(k: int, P: Point) -> Point:
    """Double-and-add scalar multiplication."""
    result = O
    addend = P
    while k:
        if k & 1:
            result = result + addend
        addend = addend + addend
        k >>= 1
    return result


# ── SHA-256 hash helper ────────────────────────────────────────────────────────

def _hash_to_int(*parts: bytes) -> int:
    """Hash concatenated byte strings to an integer mod n."""
    h = hashlib.sha256(b''.join(parts)).digest()
    return int.from_bytes(h, 'big') % _n


def _point_to_bytes(P: Point) -> bytes:
    """Compress a point to 33 bytes (02/03 prefix + x-coordinate)."""
    prefix = b'\x02' if P.y % 2 == 0 else b'\x03'
    return prefix + P.x.to_bytes(32, 'big')


# ── Exercise (a): ECDSA nonce reuse attack ─────────────────────────────────────

def ecdsa_sign_with_fixed_k(private_key: int, message_hash: int, k: int) -> tuple[int, int]:
    """Sign with an explicitly provided nonce (NEVER do this in production)."""
    R = k * G
    r = R.x % _n
    assert r != 0
    k_inv = _modinv(k, _n)
    s = k_inv * (message_hash + r * private_key) % _n
    assert s != 0
    return r, s


def ecdsa_nonce_reuse_attack(
    e1: int, e2: int,
    s1: int, s2: int,
    r: int,
    n: int = _n
) -> tuple[int, int]:
    """
    Recover nonce k and private key d from two ECDSA signatures
    with the same nonce (same r).

    Math:
        s1 = k⁻¹(e1 + r·d) mod n
        s2 = k⁻¹(e2 + r·d) mod n
        s1 - s2 = k⁻¹(e1 - e2) mod n
        k = (e1 - e2)(s1 - s2)⁻¹ mod n
        d = (s1·k - e1)·r⁻¹ mod n

    This is the PS3 attack: Sony reused k=constant, breaking the entire system.
    """
    if s1 == s2:
        raise ValueError("s1 == s2: messages must be different")

    k = (e1 - e2) * pow(s1 - s2, -1, n) % n
    d = (s1 * k - e1) * pow(r, -1, n) % n
    return k, d


def demonstrate_ecdsa_nonce_reuse():
    """Full end-to-end demonstration of the nonce reuse attack."""
    print("=" * 65)
    print("Exercise (a): ECDSA Nonce Reuse Key Recovery")
    print("=" * 65)

    # Alice's keypair
    d_alice = secrets.randbelow(_n - 1) + 1
    Y_alice = d_alice * G

    # Two messages (simulated hashes; in practice these are SHA-256 of the message)
    m1 = b"Transaction: pay Bob 1 BTC"
    m2 = b"Transaction: pay Charlie 2 BTC"
    e1 = int.from_bytes(hashlib.sha256(m1).digest(), 'big') % _n
    e2 = int.from_bytes(hashlib.sha256(m2).digest(), 'big') % _n

    # VULNERABILITY: reuse the same nonce for both signatures
    k_bad = secrets.randbelow(_n - 1) + 1  # normally this must be unique per sig

    r, s1 = ecdsa_sign_with_fixed_k(d_alice, e1, k_bad)
    _, s2 = ecdsa_sign_with_fixed_k(d_alice, e2, k_bad)

    # Attacker observes (r, s1, s2, e1, e2) — all are public
    print(f"\n  Public info: r={r:#010x}..., s1={s1:#010x}..., s2={s2:#010x}...")
    print(f"  Attacker runs nonce reuse recovery...")

    recovered_k, recovered_d = ecdsa_nonce_reuse_attack(e1, e2, s1, s2, r)

    assert recovered_k == k_bad,   "Nonce recovery failed!"
    assert recovered_d == d_alice, "Private key recovery failed!"

    # Verify recovered public key matches
    recovered_Y = recovered_d * G
    assert recovered_Y == Y_alice, "Recovered public key mismatch!"

    print(f"  [✓] Recovered nonce k = {recovered_k:#010x}...")
    print(f"  [✓] Recovered private key d = {recovered_d:#010x}...")
    print(f"  [✓] d·G matches Alice's public key Y_alice")
    print(f"\n  LESSON: NEVER reuse ECDSA nonces.")
    print(f"          Use RFC 6979 deterministic k: k = HMAC-DRBG(d, H(m))")
    print(f"          PS3, Sony/fail0verflow (2010), weak Bitcoin keys (2012-2016)")


# ── Exercise (b): Schnorr ZKP test suite ──────────────────────────────────────

class SchnorrProof:
    """
    Schnorr proof of knowledge of discrete logarithm.
    Interactive protocol made non-interactive via Fiat-Shamir.

    Prover knows: secret x such that Y = x·G
    Proof: (Y, A, s) where A = r·G, s = r + H(G||Y||A)·x mod n
    Verify: s·G == A + H(G||Y||A)·Y
    """

    def __init__(self, G: Point = G, n: int = _n):
        self.G = G
        self.n = n

    def _challenge(self, Y: Point, A: Point) -> int:
        """Fiat-Shamir challenge: hash the statement and commitment."""
        data = (
            _point_to_bytes(self.G)
            + _point_to_bytes(Y)
            + _point_to_bytes(A)
        )
        return _hash_to_int(data)

    def prove(self, secret: int) -> tuple[Point, Point, int]:
        """
        Generate a Schnorr proof.
        Returns (Y, A, s):
          Y = secret·G  (public key / statement)
          A = r·G       (commitment, for random nonce r)
          s = r + c·secret mod n  (response)
        """
        Y = secret * self.G
        r = secrets.randbelow(self.n - 1) + 1   # random nonce
        A = r * self.G
        c = self._challenge(Y, A)
        s = (r + c * secret) % self.n
        return Y, A, s

    def verify(self, Y: Point, A: Point, s: int) -> bool:
        """
        Verify a Schnorr proof.
        Check: s·G == A + c·Y  where c = H(G||Y||A)
        """
        if s <= 0 or s >= self.n:
            return False
        c = self._challenge(Y, A)
        lhs = s * self.G
        rhs = A + (c * Y)
        return lhs == rhs


def schnorr_complete_test_suite():
    """
    Complete Schnorr test suite as specified by the exercise.
    Tests: basic verify, tamper s, tamper A, tamper Y, edge cases.
    """
    print("\n" + "=" * 65)
    print("Exercise (b): Schnorr Proof System — Complete Test Suite")
    print("=" * 65)

    schnorr = SchnorrProof()

    # Step 1: Generate random secret
    secret = secrets.randbelow(_n - 1) + 1
    print(f"\n  [1] Random secret: {secret.bit_length()} bits")

    # Step 2: Create proof
    Y, A, s = schnorr.prove(secret)
    print(f"  [2] Proof created: Y={Y.x:#010x}..., A={A.x:#010x}..., s={s:#010x}...")

    # Step 3: Verify proof succeeds
    assert schnorr.verify(Y, A, s), "Proof should verify!"
    print(f"  [3] ✓ Valid proof VERIFIES")

    # Step 4: Tamper with s (+1) → should fail
    s_tampered = (s + 1) % _n
    assert not schnorr.verify(Y, A, s_tampered), "Tampered s+1 should FAIL"
    print(f"  [4] ✓ Tampered s+1 REJECTED (soundness)")

    # Additional s tamperings
    for delta, label in [(-1, "s-1"), (2, "s+2"), (100, "s+100"), (_n//3, "s+n/3")]:
        assert not schnorr.verify(Y, A, (s + delta) % _n), f"Tamper {label} should fail"
    print(f"       ✓ Multiple s-tamperings all REJECTED")

    # Tamper commitment A → should fail
    _, A_wrong, _ = schnorr.prove(secrets.randbelow(_n - 1) + 1)
    assert not schnorr.verify(Y, A_wrong, s), "Wrong A should fail"
    print(f"       ✓ Wrong commitment A REJECTED")

    # Tamper public key Y → should fail
    Y_wrong = (secret + 1) * G   # different secret's public key
    assert not schnorr.verify(Y_wrong, A, s), "Wrong Y should fail"
    print(f"       ✓ Wrong public key Y REJECTED")

    # Edge case: secret = 1
    Y1, A1, s1_proof = schnorr.prove(1)
    assert schnorr.verify(Y1, A1, s1_proof)
    print(f"       ✓ Edge case: secret = 1 works")

    # Edge case: secret = n-1 (maximum valid value)
    Yn, An, sn = schnorr.prove(_n - 1)
    assert schnorr.verify(Yn, An, sn)
    print(f"       ✓ Edge case: secret = n-1 works")

    # Two proofs for the same secret are different (randomized nonce)
    _, A_a, s_a = schnorr.prove(secret)
    _, A_b, s_b = schnorr.prove(secret)
    assert (A_a != A_b) or (s_a != s_b), "Two proofs should differ (randomized r)"
    assert schnorr.verify(Y, A_a, s_a) and schnorr.verify(Y, A_b, s_b)
    print(f"       ✓ Two proofs for same secret are DISTINCT (random nonce)")

    # Proof from one key doesn't verify for another key
    secret2 = secrets.randbelow(_n - 1) + 1
    Y2 = secret2 * G
    assert not schnorr.verify(Y2, A, s), "Proof for secret1 shouldn't verify for Y2"
    print(f"       ✓ Cross-key verification correctly REJECTED")

    print(f"\n  ✓ All Schnorr tests PASSED")


# ── Exercise (c): TLS 1.3 forward secrecy analysis ────────────────────────────

def tls13_hkdf_extract(salt: bytes, ikm: bytes) -> bytes:
    """HKDF-Extract: PRK = HMAC-SHA256(salt, ikm)."""
    return hmac.new(salt, ikm, hashlib.sha256).digest()


def tls13_hkdf_expand_label(secret: bytes, label: str, context: bytes, length: int) -> bytes:
    """
    HKDF-ExpandLabel as defined in RFC 8446 §7.1.
    Label format: HkdfLabel = { length (uint16), "tls13 " + label, context }
    """
    tls13_label = b"tls13 " + label.encode()
    hkdf_label = (
        struct.pack(">H", length)        # length (2 bytes, big-endian)
        + bytes([len(tls13_label)])       # label length (1 byte)
        + tls13_label                     # label
        + bytes([len(context)])           # context length (1 byte)
        + context                         # context
    )
    # HKDF-Expand using HMAC-SHA256
    # T(1) = HMAC(secret, hkdf_label || 0x01)
    return hmac.new(secret, hkdf_label + b'\x01', hashlib.sha256).digest()[:length]


def simulate_tls13_key_schedule(
    shared_secret: bytes,
    client_hello_hash: bytes,
    server_hello_hash: bytes,
):
    """
    Simulate the TLS 1.3 key schedule (RFC 8446 §7.1).
    Demonstrates that the certificate private key is not involved.
    """
    print("\n" + "=" * 65)
    print("Exercise (c): TLS 1.3 Key Schedule Simulation")
    print("=" * 65)

    zero_32 = b'\x00' * 32

    # 1. Early secret (no PSK in this example)
    early_secret = tls13_hkdf_extract(zero_32, zero_32)
    print(f"\n  early_secret = HKDF-Extract(0, 0) = {early_secret.hex()[:16]}...")

    # 2. Derive secret before handshake
    derived_secret = tls13_hkdf_expand_label(early_secret, "derived", b'', 32)
    print(f"  derived_secret                     = {derived_secret.hex()[:16]}...")

    # 3. Handshake secret (mixes in the ECDH shared secret)
    handshake_secret = tls13_hkdf_extract(derived_secret, shared_secret)
    print(f"  handshake_secret = HKDF-Extract(derived, shared_secret)")
    print(f"                   = {handshake_secret.hex()[:16]}...")
    print(f"    ← shared_secret = ECDH(client_ephemeral, server_ephemeral)")
    print(f"      Certificate private key is NOT used here")

    # 4. Client/server handshake traffic secrets
    transcript = client_hello_hash + server_hello_hash
    c_hs = tls13_hkdf_expand_label(handshake_secret, "c hs traffic", transcript, 32)
    s_hs = tls13_hkdf_expand_label(handshake_secret, "s hs traffic", transcript, 32)
    print(f"  client_hs_traffic_secret           = {c_hs.hex()[:16]}...")
    print(f"  server_hs_traffic_secret           = {s_hs.hex()[:16]}...")

    # 5. Master secret
    derived2 = tls13_hkdf_expand_label(handshake_secret, "derived", b'', 32)
    master_secret = tls13_hkdf_extract(derived2, zero_32)
    print(f"  master_secret = HKDF-Extract(derived2, 0)")
    print(f"               = {master_secret.hex()[:16]}...")

    # 6. Application traffic secrets
    full_transcript = transcript + b"server_certificate_verify"
    c_ap = tls13_hkdf_expand_label(master_secret, "c ap traffic", full_transcript, 32)
    s_ap = tls13_hkdf_expand_label(master_secret, "s ap traffic", full_transcript, 32)
    print(f"  client_app_traffic_secret          = {c_ap.hex()[:16]}...")
    print(f"  server_app_traffic_secret          = {s_ap.hex()[:16]}...")
    print(f"  (These derive the actual AES-GCM keys that encrypt all application data)")

    # 7. Demonstrate: changing shared_secret changes everything
    print(f"\n  --- Forward Secrecy Demonstration ---")
    fake_shared = secrets.token_bytes(32)  # different ECDH shared secret
    hs2 = tls13_hkdf_extract(derived_secret, fake_shared)
    derived2b = tls13_hkdf_expand_label(hs2, "derived", b'', 32)
    ms2 = tls13_hkdf_extract(derived2b, zero_32)
    c_ap2 = tls13_hkdf_expand_label(ms2, "c ap traffic", full_transcript, 32)
    print(f"  With different ephemeral keys, app traffic key changes:")
    print(f"    Session 1: {c_ap.hex()[:16]}...")
    print(f"    Session 2: {c_ap2.hex()[:16]}...")
    assert c_ap != c_ap2
    print(f"  Each session has a unique key — stealing cert key reveals NEITHER")

    # 8. The cert private key only signs the transcript
    print(f"\n  Certificate private key usage:")
    print(f"    CertificateVerify = ECDSA_Sign(cert_private_key,")
    print(f"                          SHA256('TLS 1.3, server CertificateVerify' || transcript))")
    print(f"    → Used ONLY for authentication, NOT key derivation")
    print(f"    → Stealing cert key: can forge future CertificateVerify")
    print(f"    → Cannot: compute shared_secret (requires ephemeral private key)")
    print(f"    → Cannot: decrypt past sessions")
    print(f"\n  CONCLUSION: Forward secrecy is established at the moment")
    print(f"  server_ephemeral_private is securely deleted after ECDH.")


def forward_secrecy_analysis():
    """Demonstrate the forward secrecy argument concisely."""
    print("\n" + "=" * 65)
    print("Forward Secrecy Summary")
    print("=" * 65)

    print("""
  Stolen: cert_private_key (long-term identity key)
  NOT stolen: server_ephemeral_private (session key, deleted after handshake)

  Key schedule inputs:
    early_secret    ← PSK (or 0 for fresh session)
    shared_secret   ← ECDH(server_ephemeral_private, client_ephemeral_public)
    master_secret   ← derived from handshake_secret(shared_secret)
    app_key         ← derived from master_secret

  cert_private_key is used only for:
    CertificateVerify = Sign(cert_key, transcript_hash)

  To decrypt past session, attacker needs:
    shared_secret = ECDH(server_ephemeral_private, ...)
    But server_ephemeral_private is DELETED after use.
    Recovering it from server_ephemeral_public requires solving ECDLP.

  What DOES break with stolen cert_private_key:
    ✗ Authentication in future sessions (attacker can impersonate server)
    ✗ Active MITM: present fake cert, intercept future sessions
    ✓ Past sessions: SAFE (forward secrecy holds)
    ✓ 1-RTT data: SAFE
    ✗ 0-RTT data: UNSAFE if PSK is also compromised (no forward secrecy in 0-RTT)
    """)


# ── Main ───────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    demonstrate_ecdsa_nonce_reuse()
    schnorr_complete_test_suite()

    # TLS 1.3 key schedule simulation
    shared_secret   = secrets.token_bytes(32)   # simulated ECDH output
    client_hello_h  = hashlib.sha256(b"ClientHello").digest()
    server_hello_h  = hashlib.sha256(b"ServerHello").digest()
    simulate_tls13_key_schedule(shared_secret, client_hello_h, server_hello_h)
    forward_secrecy_analysis()
```

---

## 4. Common Wrong Approaches and Why They Fail

### Wrong Approach 1: Thinking ECDSA Nonce Reuse Requires Identical Messages

**Mistake:** "The attack only works if both messages are identical."

**Why it fails:** The attack requires `s1 ≠ s2`, which requires `e1 ≠ e2`, which requires `m1 ≠ m2`. If the messages are identical, `s1 = s2` and the two equations become redundant — no information about `k`. But with **different messages and same nonce**, the attack is fully algebraic. Sony's PS3 had `k = constant` across ALL signatures with different games/executables, making every pair of signatures exploitable.

### Wrong Approach 2: Using HKDF Instead of Random Nonces for ECDSA

**Mistake:** "Derive the nonce as HMAC(secret_key, message) — this is deterministic so it must be safe."

**Why it fails:** Almost — this is close to RFC 6979, which derives `k = HMAC-DRBG(d, H(m) || V || 0x00 || additional_data)`. The key point: the nonce derivation must include the private key `d` so that even with the same message, a different signing key produces a different `k`. A naive `HMAC(constant_key, m)` without `d` produces the same `k` across all signing keys for the same message — defeating the point. RFC 6979 is the correct approach, not "any deterministic function of the message."

### Wrong Approach 3: Believing Schnorr's Zero-Knowledge Property Means the Proof Hides the Public Key

**Mistake:** "The Schnorr proof is zero-knowledge, so the verifier learns nothing about Y = x·G."

**Why it fails:** Zero-knowledge means the verifier learns nothing about the **secret** `x` beyond what's implied by knowing `Y`. The public key `Y` is part of the statement being proven and is public. The verifier already knows `Y` before the proof. ZK means: given `Y`, no additional information about `x` is revealed. The proof `(A, s)` is simulable without `x`, which is the formal ZK property.

### Wrong Approach 4: Thinking TLS 1.3 0-RTT Is Safe Because the PSK Was Derived Securely

**Mistake:** "The PSK is derived from a past session's master secret, which used ephemeral ECDH — so 0-RTT data has forward secrecy."

**Why it fails:** The PSK is stored as a session ticket on the client and encrypted by the server's session ticket key (a symmetric key the server keeps long-term). If the server's session ticket key is compromised (which is separate from the certificate private key), an attacker can:
1. Decrypt session tickets from recorded traffic to recover PSKs
2. Use those PSKs to decrypt 0-RTT data from recorded connections
3. Or replay the 0-RTT data to the server (no anti-replay in 0-RTT by design)

For true forward secrecy of session resumption, the server must rotate its session ticket key frequently (every hour in high-security configurations) or use stateful session IDs instead of stateless tickets.

### Wrong Approach 5: Assuming ECDH Public Keys Don't Need Validation

**Mistake:** "The client sends its public key point to the server; the server doesn't need to check if it's a valid point."

**Why it fails:** Without validation, a malicious client can send a point on a different curve (an "invalid curve attack"), which leaks bits of the server's private key through the computed "shared secret." For X25519 (Curve25519), the clamping of the scalar and the curve's cofactor-1 structure prevent this. For secp256k1 and P-256, both parties must explicitly check: `Q.y² ≡ Q.x³ + 7 (mod p)` (on curve), `Q ≠ O` (not infinity), and `n·Q = O` (in the right subgroup). RFC 8422 (TLS ECC) mandates these checks.

---

## 5. Extensions

### 1,000-Party MuSig2 Multi-Signature

MuSig2 (Maxwell, Poelstra, Wuille — 2021) allows `n` parties to collaboratively produce a single Schnorr signature that verifies against the aggregate public key `P_agg = H(L, P_1)·P_1 + H(L, P_2)·P_2 + ... + H(L, P_n)·P_n` where `L` is the list of all public keys. At `n = 1,000`:

- **Round 1:** Each party broadcasts 2 nonce commitments (R1_i, R2_i)
- **Round 2:** Each party computes the challenge c from the aggregate nonce R_agg = R1_agg + b·R2_agg, computes partial signature s_i, and broadcasts it
- **Aggregation:** s_agg = sum(s_i mod n)

Total: 2 broadcast rounds and O(n) communication. MuSig2 requires no trusted dealer and its security reduces to ECDLP hardness.

Production use: Bitcoin's Taproot (BIP 341/342) enables MuSig2-style key aggregation for threshold multi-sig wallets with single-key efficiency.

### 1TB of Signed Data

At 1TB of data to sign/verify:
- ECDSA/EdDSA only signs the hash of the data (32 bytes SHA-256 output) regardless of data size
- SHA-256 throughput on AVX2 hardware: ~2 GB/s → 1TB hashes in ~512 seconds
- **Merkle tree signatures**: sign the root of a Merkle tree over 1MB chunks. Each chunk gets an inclusion proof (O(log n) hashes). Anyone can verify any chunk independently.
- Certificate Transparency (RFC 9162) uses exactly this: a signed Merkle tree over certificate logs, enabling efficient inclusion proofs.

### Adversarial: Timing Side-Channel

Even a secure ECDSA implementation is vulnerable to timing attacks if scalar multiplication branches on secret bits. The double-and-add algorithm (`if k & 1: result = result + addend`) takes different time for different bits.

Countermeasures:
- **Montgomery ladder:** always performs both a point addition and a point doubling per bit, then selects the result with a constant-time conditional select (no branch)
- **Scalar blinding:** add a random multiple of the group order to the scalar: `k' = k + r·n` — same result, but timing depends on `r` rather than `k`
- **Point blinding:** add a random point to the base: work with `G' = G + R_random`, subtract `R_random` at the end — hides intermediate points from power analysis

OpenSSL's `ec_wNAF_mul()` uses the wNAF (windowed Non-Adjacent Form) representation plus point blinding. BoringSSL's p256 scalar multiplication is completely constant-time in assembly.

---

## 6. Real-World Production System References

### PS3 Root Key Compromise (fail0verflow, CCC 2010)

George Hotz (geohot) and fail0verflow demonstrated the nonce reuse attack on the PS3's firmware signing system at Chaos Communication Congress 2010. Sony's `lv2` hypervisor used ECDSA with secp192r1 and a constant `k` derived from a global counter that was initialized the same way on every boot. Extracting two firmware signatures with the same `r` value immediately yielded the private key via the linear algebra in this solution. Sony's subsequent lawsuit and the public release of the private key demonstrated that once deployed, broken cryptography cannot be recalled.

### Bitcoin Weak Nonce Exploits (2013–2016)

Brainflayer (2016) analyzed the Bitcoin blockchain for ECDSA signatures with repeated `r` values (indicating nonce reuse). Research by security researcher `johoe` found ~158 Bitcoin addresses with reused nonces, mostly from broken Android `SecureRandom` implementations (a kernel PRNG initialization bug on Android 4.x caused `k` values to repeat for several hours after boot). Funds from those addresses were immediately drained upon publication. The total loss was ~55 BTC (~$500 at 2013 prices).

### TLS 1.3 Key Schedule: RFC 8446 §7.1

RFC 8446 is the authoritative reference for the TLS 1.3 key schedule. Section 7.1 defines the exact HKDF-Extract and HKDF-ExpandLabel calls. Section 7.4.3 specifies the CertificateVerify message format and the context string "TLS 1.3, server CertificateVerify". The RFC explicitly states in §2.2: "In contrast to TLS 1.2, resumption in TLS 1.3 is based on PSKs... The PSK cannot be used to derive traffic keys for the previous session; it is derived from the session tickets."

### OpenSSL's ECDSA Implementation: `crypto/ec/ecdsa_ossl.c`

OpenSSL's `ossl_ecdsa_sign_sig()` uses `BN_generate_dsa_nonce()` for `k` generation, which implements RFC 6979's HMAC-DRBG. The exact implementation is in `crypto/bn/bn_dh.c`. Prior to OpenSSL 1.0.2 (2015), OpenSSL used `BN_rand_range()` — a plain random number generator — making it vulnerable on systems with weak entropy (embedded systems, early boot, VMs with predictable state).

### Signal Protocol's Double Ratchet

Signal's Double Ratchet algorithm (Marlinspike & Perrin, 2016) achieves **forward secrecy per message**: each message uses a fresh key derived from the ratchet state, and old keys are immediately deleted. It also provides **break-in recovery** (future secrecy): even if an attacker learns the current state, future messages are protected once the ratchet advances. This is a stronger property than TLS 1.3's per-session forward secrecy. The Double Ratchet is used in Signal, WhatsApp, Facebook Messenger (Secret Conversations), and Matrix.
