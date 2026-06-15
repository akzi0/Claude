# Solution: ECDSA Nonce Reuse Attack + Schnorr + TLS 1.3 Forward Secrecy (Hard Exercise — Lesson 07)

## The Question

Three parts:
- **(a)** Mathematical derivation of ECDSA nonce reuse key recovery, with Python implementation
- **(b)** Full Schnorr proof implementation with tamper test
- **(c)** Why TLS 1.3 forward secrecy prevents decrypting past sessions even with the server's certificate private key

---

## (a) ECDSA Nonce Reuse Key Recovery

### Background: How ECDSA Signing Works

ECDSA (Elliptic Curve Digital Signature Algorithm) uses a secret nonce `k` per signature:

```
1. Choose random k ∈ [1, n-1]   (n = group order of the curve)
2. Compute R = k·G               (G = generator point)
3. Set r = R.x mod n             (x-coordinate of R)
4. Compute s = k⁻¹ · (H(m) + r·d) mod n
                              (d = private key, H(m) = hash of message)
5. Signature = (r, s)
```

If the **same k is used for two different messages** m1 and m2:
- Both signatures share the same `r` (since `R = k·G` is the same)
- `s1 = k⁻¹ · (e1 + r·d) mod n` where `e1 = H(m1)`
- `s2 = k⁻¹ · (e2 + r·d) mod n` where `e2 = H(m2)`

### Mathematical Derivation

**Step 1: Recover k from s1 and s2**

```
s1 = k⁻¹(e1 + r·d) mod n    ... (1)
s2 = k⁻¹(e2 + r·d) mod n    ... (2)

Subtract (2) from (1):
s1 - s2 = k⁻¹(e1 + r·d) - k⁻¹(e2 + r·d) mod n
         = k⁻¹(e1 - e2) mod n

Therefore:
k = (e1 - e2) · (s1 - s2)⁻¹ mod n
```

**Step 2: Recover private key d from k**

```
From equation (1):
s1 · k = e1 + r·d mod n
r·d = s1·k - e1 mod n
d = (s1·k - e1) · r⁻¹ mod n
```

**This is all the math needed.** An attacker observing two signatures `(r, s1)` and `(r, s2)` on messages `m1, m2` can compute `k` and then `d` using only modular arithmetic.

### Why This Broke Real Systems

- **Sony PlayStation 3 (2010):** RPCS3 reverse engineers discovered Sony used the same `k` for every firmware signature. The constant `k` made all signatures use the same `r`, and a single pair of messages leaked `k` and then `d`.
- **Bitcoin wallets:** Several early Android Bitcoin wallets used a broken `SecureRandom` that produced repeated `k` values, leaking private keys to anyone who recorded two transactions.

### Python Implementation

```python
"""
ECDSA Nonce Reuse Key Recovery
Implements secp256k1 curve operations from scratch for educational purposes.
DO NOT use in production — use libsecp256k1 or cryptography library instead.
"""
import hashlib
import secrets
from dataclasses import dataclass
from typing import Optional, Tuple


@dataclass
class Point:
    """Affine point on an elliptic curve."""
    x: Optional[int]
    y: Optional[int]
    
    @classmethod
    def infinity(cls) -> 'Point':
        return cls(None, None)
    
    def is_infinity(self) -> bool:
        return self.x is None and self.y is None
    
    def __eq__(self, other) -> bool:
        return self.x == other.x and self.y == other.y


class Curve:
    """Weierstrass elliptic curve: y² = x³ + ax + b (mod p)"""
    
    def __init__(self, p: int, a: int, b: int):
        self.p = p
        self.a = a
        self.b = b
    
    def point_add(self, P: Point, Q: Point) -> Point:
        if P.is_infinity(): return Q
        if Q.is_infinity(): return P
        
        if P.x == Q.x and P.y != Q.y:
            return Point.infinity()  # P = -Q
        
        if P == Q:
            # Point doubling
            if P.y == 0:
                return Point.infinity()
            lam = (3 * P.x * P.x + self.a) * pow(2 * P.y, -1, self.p) % self.p
        else:
            # Point addition
            lam = (Q.y - P.y) * pow(Q.x - P.x, -1, self.p) % self.p
        
        x3 = (lam * lam - P.x - Q.x) % self.p
        y3 = (lam * (P.x - x3) - P.y) % self.p
        return Point(x3, y3)
    
    def scalar_mul(self, k: int, P: Point) -> Point:
        """Double-and-add scalar multiplication."""
        result = Point.infinity()
        addend = P
        while k:
            if k & 1:
                result = self.point_add(result, addend)
            addend = self.point_add(addend, addend)
            k >>= 1
        return result


# secp256k1 parameters (used by Bitcoin, Ethereum)
_p = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F
_a = 0
_b = 7
_n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
_Gx = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798
_Gy = 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8

secp256k1 = Curve(_p, _a, _b)
G = Point(_Gx, _Gy)


def hash_message(msg: bytes) -> int:
    """SHA-256 hash of message, returned as integer."""
    return int.from_bytes(hashlib.sha256(msg).digest(), 'big') % _n


def ecdsa_sign(msg: bytes, private_key: int, k: int) -> Tuple[int, int]:
    """Sign msg with private_key using nonce k. Returns (r, s)."""
    e = hash_message(msg)
    R = secp256k1.scalar_mul(k, G)
    r = R.x % _n
    if r == 0:
        raise ValueError("r == 0, try a different k")
    s = pow(k, -1, _n) * (e + r * private_key) % _n
    if s == 0:
        raise ValueError("s == 0, try a different k")
    return r, s


def ecdsa_verify(msg: bytes, sig: Tuple[int, int], public_key: Point) -> bool:
    """Verify ECDSA signature. Returns True if valid."""
    r, s = sig
    if not (1 <= r < _n and 1 <= s < _n):
        return False
    e = hash_message(msg)
    w = pow(s, -1, _n)
    u1 = e * w % _n
    u2 = r * w % _n
    X = secp256k1.point_add(secp256k1.scalar_mul(u1, G), secp256k1.scalar_mul(u2, public_key))
    if X.is_infinity():
        return False
    return X.x % _n == r


def ecdsa_nonce_reuse_attack(
    e1: int, e2: int,   # H(m1), H(m2)
    s1: int, s2: int,   # signature s components
    r: int,             # shared r (same nonce k)
    n: int              # group order
) -> Tuple[int, int]:
    """
    Recover nonce k and private key d from two signatures sharing the same nonce.
    
    Math:
      k = (e1 - e2) * (s1 - s2)^{-1} mod n
      d = (s1 * k - e1) * r^{-1} mod n
    """
    if s1 == s2:
        raise ValueError("s1 == s2 implies e1 == e2 — same message, nothing to recover")
    
    k = (e1 - e2) * pow(s1 - s2, -1, n) % n
    d = (s1 * k - e1) * pow(r, -1, n) % n
    return k, d


def demonstrate_nonce_reuse_attack():
    """Full demonstration: sign two messages with same k, then recover keys."""
    # Alice's keypair
    alice_private = secrets.randbelow(_n - 1) + 1
    alice_public = secp256k1.scalar_mul(alice_private, G)
    
    m1 = b"Transaction: pay Bob 1 BTC"
    m2 = b"Transaction: pay Charlie 2 BTC"
    
    # VULNERABILITY: reuse same k for both signatures
    k_reused = secrets.randbelow(_n - 1) + 1
    
    # Sign both messages
    sig1 = ecdsa_sign(m1, alice_private, k_reused)
    sig2 = ecdsa_sign(m2, alice_private, k_reused)
    
    r1, s1 = sig1
    r2, s2 = sig2
    
    # Both signatures have the same r (because same k → same R = k·G → same r = R.x)
    assert r1 == r2, "Different r values — nonce not reused!"
    
    # Verify signatures are valid
    assert ecdsa_verify(m1, sig1, alice_public), "sig1 invalid"
    assert ecdsa_verify(m2, sig2, alice_public), "sig2 invalid"
    
    # Attacker observes: (r, s1, s2, m1, m2) — all public
    e1 = hash_message(m1)
    e2 = hash_message(m2)
    
    recovered_k, recovered_d = ecdsa_nonce_reuse_attack(e1, e2, s1, s2, r1, _n)
    
    # Verify recovery
    assert recovered_k == k_reused, f"k recovery failed: {recovered_k} != {k_reused}"
    assert recovered_d == alice_private, f"d recovery failed"
    
    recovered_public = secp256k1.scalar_mul(recovered_d, G)
    assert recovered_public == alice_public, "Recovered key doesn't match public key"
    
    print("ECDSA Nonce Reuse Attack - SUCCESSFUL")
    print(f"  k (original):  {k_reused:#066x}")
    print(f"  k (recovered): {recovered_k:#066x}")
    print(f"  d (original):  {alice_private:#066x}")
    print(f"  d (recovered): {recovered_d:#066x}")
    print(f"  Result: Alice's private key fully recovered from two signatures!")
    print()
    print("  FIX: Use RFC 6979 deterministic k (never repeats, never random)")


# ─── DEFENSE: RFC 6979 Deterministic Nonce ──────────────────────────────────────

def rfc6979_nonce(private_key: int, msg_hash: int, n: int) -> int:
    """
    RFC 6979: Deterministic nonce generation.
    k = HMAC-DRBG(private_key, H(m))
    
    Properties:
    - Deterministic: same key + same message → always same k
    - Never repeats for different messages (HMAC-based PRF)
    - No RNG dependency: eliminates weak/biased RNG vulnerabilities
    
    Real implementation: use python-ecdsa library's built-in RFC 6979 support.
    """
    import hmac
    # Simplified RFC 6979 (full implementation omitted for brevity)
    # The actual algorithm uses HMAC-DRBG with V and K state variables
    key_bytes = private_key.to_bytes(32, 'big')
    msg_bytes = msg_hash.to_bytes(32, 'big')
    h = hmac.new(key_bytes, msg_bytes, hashlib.sha256).digest()
    k = int.from_bytes(h, 'big') % (n - 1) + 1
    return k


if __name__ == "__main__":
    demonstrate_nonce_reuse_attack()
```

---

## (b) Schnorr Proof of Knowledge — Full Implementation + Tamper Test

### How Schnorr Proof Works

A Schnorr proof allows a prover to convince a verifier they know `x` (the discrete log of `Y = x·G`) without revealing `x`:

```
Setup: Public key Y = x·G (prover knows x, verifier knows Y)

Prove:
  1. Prover chooses random r ∈ [1, n-1]
  2. Prover computes commitment A = r·G
  3. Verifier sends challenge c = H(Y, A)  [or non-interactive via Fiat-Shamir]
  4. Prover computes response s = r + c·x mod n
  5. Prover sends (A, s) to verifier

Verify:
  Check: s·G == A + c·Y
  
  Why this works:
    s·G = (r + c·x)·G = r·G + c·x·G = A + c·Y ✓
    
  Why it's zero-knowledge:
    Verifier sees (A, s). They can't compute x from s = r + c·x
    because r is unknown and c is computed from A (which hides r).
    
  Why it's sound (binding):
    If prover can answer two different challenges c ≠ c' for the same A:
    s  = r + c·x mod n
    s' = r + c'·x mod n
    s - s' = (c - c')·x mod n
    x = (s - s') · (c - c')⁻¹ mod n
    → Prover who can forge two responses KNOWS x. Soundness holds.
```

```python
"""Schnorr Non-Interactive Zero-Knowledge Proof (Fiat-Shamir heuristic)"""
import hashlib
import secrets
from typing import Tuple


class SchnorrProof:
    """
    Schnorr proof of knowledge of discrete log.
    Non-interactive via Fiat-Shamir: c = H(G || Y || A).
    """
    
    def __init__(self, curve: Curve, G: Point, n: int):
        self.curve = curve
        self.G = G
        self.n = n
    
    def _challenge(self, Y: Point, A: Point) -> int:
        """Fiat-Shamir: compute challenge c = H(Y || A)."""
        data = (
            Y.x.to_bytes(32, 'big') +
            Y.y.to_bytes(32, 'big') +
            A.x.to_bytes(32, 'big') +
            A.y.to_bytes(32, 'big')
        )
        return int.from_bytes(hashlib.sha256(data).digest(), 'big') % self.n
    
    def prove(self, secret: int) -> Tuple[Point, Point, int]:
        """
        Generate a Schnorr proof of knowledge of secret x where Y = x·G.
        Returns (Y, A, s) where:
          Y = public key = x·G
          A = commitment = r·G (randomized each call)
          s = response = r + c·x mod n
        """
        if not (1 <= secret < self.n):
            raise ValueError(f"Secret must be in [1, n-1], got {secret}")
        
        Y = self.curve.scalar_mul(secret, self.G)     # Public key
        r = secrets.randbelow(self.n - 1) + 1         # Random nonce
        A = self.curve.scalar_mul(r, self.G)           # Commitment
        c = self._challenge(Y, A)                      # Challenge (Fiat-Shamir)
        s = (r + c * secret) % self.n                 # Response
        
        return Y, A, s
    
    def verify(self, Y: Point, A: Point, s: int) -> bool:
        """
        Verify a Schnorr proof.
        Returns True iff the prover knows x such that Y = x·G.
        
        Check: s·G == A + c·Y
        """
        if Y.is_infinity() or A.is_infinity():
            return False
        if not (0 < s < self.n):
            return False
        
        c = self._challenge(Y, A)
        
        lhs = self.curve.scalar_mul(s, self.G)          # s·G
        rhs = self.curve.point_add(                      # A + c·Y
            A,
            self.curve.scalar_mul(c, Y)
        )
        
        return lhs == rhs


def schnorr_complete_test_suite():
    """
    Complete test suite for the Schnorr proof system.
    Tests correctness, tamper resistance, and edge cases.
    """
    schnorr = SchnorrProof(secp256k1, G, _n)
    passed = 0
    failed = 0
    
    def check(condition: bool, name: str):
        nonlocal passed, failed
        if condition:
            passed += 1
            print(f"  [PASS] {name}")
        else:
            failed += 1
            print(f"  [FAIL] {name}")
    
    print("=== Schnorr Proof System Test Suite ===\n")
    
    # Test 1: Generate valid proof
    secret = secrets.randbelow(_n - 1) + 1
    Y, A, s = schnorr.prove(secret)
    check(schnorr.verify(Y, A, s), "Valid proof verifies")
    
    # Test 2: Tamper s by +1
    check(not schnorr.verify(Y, A, (s + 1) % _n), "Tampered s+1 rejected")
    
    # Test 3: Tamper s by -1
    check(not schnorr.verify(Y, A, (s - 1) % _n), "Tampered s-1 rejected")
    
    # Test 4: Tamper s by +n/2
    check(not schnorr.verify(Y, A, (s + _n // 2) % _n), "Tampered s+n/2 rejected")
    
    # Test 5: Wrong commitment A (from different proof)
    _, A2, _ = schnorr.prove(secrets.randbelow(_n - 1) + 1)
    check(not schnorr.verify(Y, A2, s), "Wrong commitment A rejected")
    
    # Test 6: Wrong public key Y
    wrong_Y = secp256k1.scalar_mul(secret + 1, G)
    check(not schnorr.verify(wrong_Y, A, s), "Wrong public key Y rejected")
    
    # Test 7: s = 0 (invalid)
    check(not schnorr.verify(Y, A, 0), "s=0 rejected")
    
    # Test 8: s = n (out of range)
    check(not schnorr.verify(Y, A, _n), "s=n (out of range) rejected")
    
    # Test 9: Edge case — secret = 1
    Y1, A1, s1 = schnorr.prove(1)
    check(schnorr.verify(Y1, A1, s1), "secret=1 (minimum) works")
    
    # Test 10: Edge case — secret = n-1 (maximum)
    Yn, An, sn = schnorr.prove(_n - 1)
    check(schnorr.verify(Yn, An, sn), "secret=n-1 (maximum) works")
    
    # Test 11: Two proofs for same secret are different (randomized A)
    _, A_a, s_a = schnorr.prove(secret)
    _, A_b, s_b = schnorr.prove(secret)
    check(
        (A_a != A_b) and schnorr.verify(Y, A_a, s_a) and schnorr.verify(Y, A_b, s_b),
        "Multiple proofs for same secret are distinct and both valid"
    )
    
    # Test 12: Cross-verification (proof for secret1 fails with Y for secret2)
    secret2 = secrets.randbelow(_n - 1) + 1
    Y2, A2x, s2 = schnorr.prove(secret2)
    check(not schnorr.verify(Y, A2x, s2), "Proof for secret2 fails verification with Y1")
    
    # Test 13: Verify proof cannot be replayed with different Y
    Y_wrong = secp256k1.scalar_mul(42, G)
    check(not schnorr.verify(Y_wrong, A, s), "Proof cannot be replayed with different public key")
    
    print(f"\nResults: {passed} passed, {failed} failed")
    if failed:
        print("WARNING: Some tests failed — check the implementation!")
    else:
        print("All Schnorr tests PASSED ✓")
    
    return failed == 0


if __name__ == "__main__":
    schnorr_complete_test_suite()
```

**Why Tampering Always Fails:**

The soundness of Schnorr relies on the binding property of the Fiat-Shamir challenge:
- Changing `s` to `s'`: the check `s'·G == A + c·Y` fails because `(s' - s)·G ≠ 0` in general.
- Changing `A` to `A'`: this changes the challenge `c' = H(Y, A') ≠ c`, so the prover would need `s' = r' + c'·x` for a new `r'` they don't have.
- The only way to generate a valid `(A, s)` is to know `x` and compute `s = r + c·x`.

---

## (c) TLS 1.3 Forward Secrecy — Why Past Sessions Cannot Be Decrypted

### The Question

If an attacker steals the TLS 1.3 server's **certificate private key** today, can they decrypt sessions recorded in the past?

**Answer: No.** Forward secrecy (also called Perfect Forward Secrecy, PFS) ensures past sessions remain secure.

### The TLS 1.3 Key Schedule

```
TLS 1.3 (RFC 8446) handshake key derivation:

Client                          Server
─────                          ──────
ClientHello                 →
  key_share: client_ephem_pub  ← ephemeral ECDH key, generated fresh for this session

                             ←  ServerHello
                                  key_share: server_ephem_pub  ← also fresh, ephemeral

Both sides compute:
  shared_secret = ECDH(client_ephem_priv, server_ephem_pub)
               = ECDH(server_ephem_priv, client_ephem_pub)

Key schedule (HKDF-based):
  early_secret      = HKDF-Extract(0, PSK_or_0)
  handshake_secret  = HKDF-Extract(
                        HKDF-Expand(early_secret, "derived"),
                        shared_secret        ← THE KEY FACT: derived from ephemeral DH
                      )
  master_secret     = HKDF-Extract(
                        HKDF-Expand(handshake_secret, "derived"),
                        0
                      )
  
  client_handshake_key = HKDF-Expand(handshake_secret, "c hs traffic", H(transcript))
  server_handshake_key = HKDF-Expand(handshake_secret, "s hs traffic", H(transcript))
  client_app_key       = HKDF-Expand(master_secret,    "c ap traffic", H(transcript))
  server_app_key       = HKDF-Expand(master_secret,    "s ap traffic", H(transcript))
```

### What the Certificate Private Key Does

```python
# Certificate private key is used ONLY for authentication, NOT key derivation:

def server_authenticate(cert_private_key: int, transcript_hash: bytes) -> bytes:
    """
    CertificateVerify message: proves server owns the certificate.
    Signs the transcript hash with the certificate private key.
    """
    signature = ecdsa_sign(
        msg=transcript_hash,
        private_key=cert_private_key,
        k=rfc6979_nonce(cert_private_key, hash_message(transcript_hash), _n)
    )
    return signature  # sent in CertificateVerify message

# The cert_private_key is NOT used to compute shared_secret, 
# handshake_secret, master_secret, or any traffic key.
```

### The Forward Secrecy Property

```
Traffic keys derive from:
  shared_secret = ECDH(client_ephem_priv × server_ephem_pub)

The ephemeral private keys (client_ephem_priv, server_ephem_pub) are:
  1. Generated fresh for each session (never reused)
  2. Stored only in RAM during the handshake
  3. ZEROED OUT immediately after shared_secret is computed

Therefore:
  To decrypt the session, attacker needs shared_secret.
  To compute shared_secret, attacker needs at least one ephemeral private key.
  Ephemeral private keys are deleted — gone forever after the handshake.

  Certificate private key ≠ ephemeral private key.
  
  Stealing cert_private_key allows:
  ✓ Forge CertificateVerify → impersonate the server in NEW sessions
  ✓ Perform active MITM against future connections
  
  But NOT:
  ✗ Compute shared_secret for past sessions (ECDLP hardness)
  ✗ Derive past handshake_secret or master_secret
  ✗ Decrypt any recorded past traffic
```

### When Forward Secrecy Is Established

Forward secrecy is established at the **precise moment** both ephemeral private keys are deleted:

```
Timeline:
  t=0: ServerHello sent, server_ephem_pub broadcast
  t=1: shared_secret computed from (server_ephem_priv × client_ephem_pub)
  t=2: server_ephem_priv ZEROED (memset(private_key_bytes, 0, 32))
       client_ephem_priv ZEROED on client side
  t=3: ← FORWARD SECRECY ESTABLISHED HERE
       All subsequent communication is encrypted with keys derived from
       the now-irrecoverable shared_secret.
```

In OpenSSL/BoringSSL: `EC_KEY_free()` calls `OPENSSL_cleanse()` on the private key bytes.

### The 0-RTT Exception

TLS 1.3 supports **0-RTT early data** (resumption using a PSK from a previous session ticket):

```
0-RTT early_data encrypted with:
  early_secret = HKDF-Extract(0, PSK)
                                 ↑
                         This is NOT ephemeral! It's a stored ticket.
```

If the PSK (server session ticket key) is compromised:
- 0-RTT data from sessions using that PSK can be decrypted
- 0-RTT data is also vulnerable to **replay attacks** (no per-session proof of freshness)

**Recommendation:** Disable 0-RTT for sensitive operations (financial transactions, authentication). Use 1-RTT for full forward secrecy.

### Python Simulation of TLS 1.3 Key Schedule

```python
"""
Simplified TLS 1.3 key schedule simulation.
Shows exactly why cert private key cannot decrypt past sessions.
"""
import hmac
import hashlib
import secrets


def hkdf_extract(salt: bytes, ikm: bytes) -> bytes:
    """HKDF-Extract: PRK = HMAC-Hash(salt, IKM)"""
    return hmac.new(salt, ikm, hashlib.sha256).digest()


def hkdf_expand_label(secret: bytes, label: str, context: bytes, length: int) -> bytes:
    """HKDF-Expand-Label as defined in RFC 8446 §7.1"""
    tls_label = b"tls13 " + label.encode()
    hkdf_label = (
        length.to_bytes(2, 'big') +
        len(tls_label).to_bytes(1, 'big') + tls_label +
        len(context).to_bytes(1, 'big') + context
    )
    # HKDF-Expand
    t = b""
    okm = b""
    for i in range(1, (length // 32) + 2):
        t = hmac.new(secret, t + hkdf_label + bytes([i]), hashlib.sha256).digest()
        okm += t
    return okm[:length]


def derive(secret: bytes, label: str, context: bytes = b"") -> bytes:
    """TLS 1.3 Derive-Secret"""
    return hkdf_expand_label(secret, label, context, 32)


def simulate_tls13_handshake(cert_private_key: int):
    """
    Simulate TLS 1.3 key derivation.
    Demonstrates that cert_private_key is NOT part of key derivation.
    """
    # Generate ephemeral ECDH keys (fresh per session)
    client_ephem_priv = secrets.randbelow(_n - 1) + 1
    server_ephem_priv = secrets.randbelow(_n - 1) + 1
    
    client_ephem_pub = secp256k1.scalar_mul(client_ephem_priv, G)
    server_ephem_pub = secp256k1.scalar_mul(server_ephem_priv, G)
    
    # ECDH key exchange
    shared_point = secp256k1.scalar_mul(client_ephem_priv, server_ephem_pub)
    assert shared_point == secp256k1.scalar_mul(server_ephem_priv, client_ephem_pub)
    shared_secret = shared_point.x.to_bytes(32, 'big')
    
    # TLS 1.3 Key Schedule
    zero_key = b'\x00' * 32
    early_secret = hkdf_extract(zero_key, zero_key)  # No PSK
    handshake_secret = hkdf_extract(
        derive(early_secret, "derived"),
        shared_secret   # ← ONLY ephemeral DH secret here
    )
    master_secret = hkdf_extract(
        derive(handshake_secret, "derived"),
        zero_key
    )
    
    # Traffic keys
    transcript_hash = hashlib.sha256(b"fake_transcript").digest()
    client_key = hkdf_expand_label(master_secret, "c ap traffic", transcript_hash, 32)
    server_key = hkdf_expand_label(master_secret, "s ap traffic", transcript_hash, 32)
    
    print("TLS 1.3 Key Derivation Simulation:")
    print(f"  cert_private_key:  {cert_private_key:#066x}")
    print(f"  client_ephem_priv: {client_ephem_priv:#066x}")
    print(f"  server_ephem_priv: {server_ephem_priv:#066x}")
    print(f"  shared_secret:     {int.from_bytes(shared_secret,'big'):#066x}")
    print(f"  client_app_key:    {client_key.hex()}")
    print(f"  server_app_key:    {server_key.hex()}")
    
    # Now: "delete" the ephemeral keys (simulate zeroing)
    client_ephem_priv_bytes = bytearray(client_ephem_priv.to_bytes(32, 'big'))
    for i in range(len(client_ephem_priv_bytes)):
        client_ephem_priv_bytes[i] = 0  # Zero out
    server_ephem_priv_bytes = bytearray(server_ephem_priv.to_bytes(32, 'big'))
    for i in range(len(server_ephem_priv_bytes)):
        server_ephem_priv_bytes[i] = 0
    
    print("\nAfter deleting ephemeral keys:")
    print(f"  client_ephem_priv (zeroed): {client_ephem_priv_bytes.hex()}")
    print(f"  server_ephem_priv (zeroed): {server_ephem_priv_bytes.hex()}")
    print()
    
    # Attacker has cert_private_key and the public ephemeral keys.
    # Can they derive shared_secret? NO — ECDLP is hard.
    print("Attacker attempt to recover session key:")
    print(f"  Has: cert_private_key = {cert_private_key:#066x}")
    print(f"  Has: client_ephem_pub.x = {client_ephem_pub.x:#066x}")
    print(f"  Has: server_ephem_pub.x = {server_ephem_pub.x:#066x}")
    print(f"  Needs: shared_secret = ECDLP(server_ephem_pub, G) → INFEASIBLE (2^128 security)")
    print()
    print("  CONCLUSION: Past sessions CANNOT be decrypted.")
    print("  The cert key is used only to sign the transcript (authentication).")
    print("  ALL traffic keys derive ONLY from the ephemeral DH shared secret.")

    return client_key, server_key


# Run the simulation
if __name__ == "__main__":
    cert_priv = secrets.randbelow(_n - 1) + 1
    simulate_tls13_handshake(cert_priv)
```

---

## Common Mistakes and Traps

### ECDSA Nonce Reuse
1. **"The signatures look different so the nonce must be different."** No — different messages with the same `k` produce different `s` values but the same `r` value. Always check if `r1 == r2` in signature logs.
2. **"I'll use a random seed."** A biased or weak RNG is as bad as reuse. Use RFC 6979 (deterministic from private key + message hash) to guarantee no repeats.
3. **"The attack needs both messages."** True — but attackers often can choose m2 (e.g., by tricking the signer into signing an attacker-chosen document).

### Schnorr
4. **"The proof is sound because the challenge is random."** Soundness relies on the challenge being chosen AFTER the commitment A is fixed. In interactive protocols this is guaranteed. In Fiat-Shamir (non-interactive), the hash H(Y, A) plays the same role — but requires the random oracle model assumption.
5. **"I can reuse the nonce r for efficiency."** No — if `r` is reused across two proofs with different secrets, both secrets can be extracted (same as ECDSA nonce reuse).

### TLS 1.3 Forward Secrecy
6. **"The server's cert key encrypts the traffic."** No — in TLS 1.3, the cert key is only used for the CertificateVerify signature. Traffic is encrypted with AEAD keys derived from the ephemeral DH shared secret.
7. **"Forward secrecy means I can't MITM future connections."** No — stealing the cert key DOES allow MITM of future connections (you can forge CertificateVerify). FS only protects PAST sessions.
8. **"0-RTT has the same FS guarantees."** No — 0-RTT uses a PSK from a session ticket, not ephemeral DH. It has no forward secrecy and is additionally vulnerable to replay attacks.
