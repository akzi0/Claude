# Lesson 07: Cryptography Internals — Elliptic Curves, Protocols, and Implementation

> **Prerequisite level:** Graduate applied cryptography. Assumes familiarity with number theory, group theory, and Python 3.10+. All cryptographic primitives are implemented from scratch using Python integers only—no `cryptography`, `pycryptodome`, or `nacl` imports. We compare our outputs against `hashlib` for hash functions only.
>
> **Version:** 1.15 (15 refinement iterations) | **Date:** 2026-06 | **Difficulty:** ★★★★★

All code blocks in this lesson are parts of one coherent module. A runnable version can be assembled by concatenating them in order. Requires only Python standard library:

```python
# Standard library imports — no third-party crypto packages
from dataclasses import dataclass
from typing import Optional
import secrets      # cryptographically secure RNG (wraps os.urandom)
import hashlib      # SHA-256/SHA-384 for comparison and HKDF
import hmac         # HMAC for RFC 5869 HKDF and RFC 6979 nonce derivation
```

---

## 1. Elliptic Curves from First Principles

### 1.1 The Weierstrass Equation

An elliptic curve E over a prime field F_p is the set of points (x, y) satisfying:

```
y² ≡ x³ + ax + b  (mod p)
```

together with a distinguished "point at infinity" O, subject to the **non-singularity** condition:

```
4a³ + 27b² ≢ 0  (mod p)
```

Singularity would mean the curve has a cusp or self-intersection, destroying the group structure. For secp256k1 (a=0, b=7):

```
4·0³ + 27·7² = 1323 ≢ 0  (mod p)    ✓
```

Why F_p? Because we need a finite structure where arithmetic is exact and where division is possible (every nonzero element has a multiplicative inverse, since p is prime). Over the reals, EC crypto would be trivially broken by real-number calculus.

### 1.2 The Group Law

The set E(F_p) = {(x,y) ∈ F_p² : y² ≡ x³+ax+b} ∪ {O} forms an **abelian group** under the following addition law:

**Identity:** O is the identity: P + O = O + P = P for all P.

**Inverses:** The inverse of (x, y) is (x, −y mod p). Note: (x,y) + (x,−y) = O.

**Geometric intuition (over ℝ):** Draw a line through P and Q; it intersects the curve at a third point R'; reflect R' over the x-axis to get R = P+Q. This works because a line intersects a cubic in exactly 3 points (counting multiplicity, in projective space), and reflection is a group automorphism.

**Over F_p:** We translate this geometry into modular arithmetic.

### 1.3 Point Addition (P ≠ Q)

Given P = (x_P, y_P) and Q = (x_Q, y_Q) with P ≠ Q, x_P ≠ x_Q:

```
λ = (y_Q - y_P) · (x_Q - x_P)^(-1)  mod p

x_R = λ² - x_P - x_Q  mod p
y_R = λ·(x_P - x_R) - y_P  mod p
```

**Why?** λ is the slope of line PQ. Substituting y = λ(x - x_P) + y_P into y² = x³+ax+b gives a cubic in x. By Vieta's formulas, x_P + x_Q + x_R = λ² (the sum of roots equals the negative of the x² coefficient). So x_R = λ² - x_P - x_Q.

**Special case:** If x_P = x_Q but y_P ≠ y_Q, then P = -Q and P + Q = O.

### 1.4 Point Doubling (P = Q)

For P = Q = (x_P, y_P) with y_P ≠ 0:

```
λ = (3x_P² + a) · (2y_P)^(-1)  mod p

x_R = λ² - 2x_P  mod p
y_R = λ·(x_P - x_R) - y_P  mod p
```

**Why?** We take the limit as Q → P of the secant line slope. By implicit differentiation of y² = x³+ax+b: 2y·dy = (3x²+a)·dx, so dy/dx = (3x²+a)/(2y). This is the tangent slope.

**Special case:** If y_P = 0, the tangent is vertical, so 2P = O.

### 1.5 Modular Inverse via Fermat's Little Theorem

For prime p and a ≢ 0 (mod p), Fermat's Little Theorem guarantees:

```
a^(p-1) ≡ 1  (mod p)
⟹  a · a^(p-2) ≡ 1  (mod p)
⟹  a^(-1) ≡ a^(p-2)  (mod p)
```

This means we can compute modular inverses using Python's built-in `pow(a, p-2, p)`, which uses fast modular exponentiation (O(log p) multiplications). For 256-bit p, that's ~256 modular squarings.

**Alternative:** Extended Euclidean algorithm gives the same result with fewer operations (O(log p) divisions), and is the production choice for performance-critical code.

### 1.6 Scalar Multiplication: Double-and-Add

The core operation in ECC is computing k·P for integer k and point P. Naive approach (k additions) is O(k) — completely impractical for 256-bit k.

**Double-and-add** exploits the binary representation of k:

```
k = k_{n-1}·2^{n-1} + ... + k_1·2 + k_0

k·P = k_{n-1}·(2^{n-1}·P) + ... + k_1·(2P) + k_0·P
```

Algorithm (left-to-right, MSB first):

```
R ← O
for i from n-1 downto 0:
    R ← 2R          (doubling)
    if k_i == 1:
        R ← R + P   (addition)
return R
```

Complexity: ~256 doublings + ~128 additions for a 256-bit key (on average half the bits are 1). This is **exponentially faster** than naive addition.

**Timing vulnerability:** The branch `if k_i == 1` causes measurable timing differences. A side-channel attacker monitoring power consumption or cache timing can extract bits of k. Production implementations use the **Montgomery ladder** which always performs both doubling and addition, regardless of the key bit.

### 1.7 The Discrete Logarithm Problem (ECDLP)

Given a point P (generator) and a point Q = k·P on curve E(F_p), find k.

**Why is this hard?** Unlike the integers where division is trivial, EC group elements have no "inverse" of scalar multiplication computable in polynomial time. The best known algorithms are:

- **Baby-step Giant-step (BSGS):** O(√n) time and space, where n = group order. Algorithm: precompute `m = ⌈√n⌉` "baby steps" {j·P | j=0..m-1} in a hash table, then test Q - i·(m·P) for i=0..m-1 (giant steps) for a match. For n ≈ 2²⁵⁶, this is ~2¹²⁸ operations and ~2¹²⁸ memory — completely infeasible (entire solar system couldn't store 2¹²⁸ points).
- **Pollard's rho:** O(√n) time, O(1) space (random walks with Floyd's cycle detection). Parallelizes across multiple machines — the gold standard for practical ECDLP attacks on small curves. Still O(√n) ≈ 2¹²⁸ for secp256k1.
- **Index calculus:** Works on F_p* (multiplicative group of integers mod p) in subexponential time L(1/3, c), which is why RSA needs ≥ 3072-bit moduli for 128-bit security. **No known analogue for EC groups** — the algebraic structure of elliptic curves resists the smoothness-based sieving. This is why ECC with 256-bit keys provides security equivalent to 3072-bit RSA keys.

  | Security Level | Symmetric | RSA/DH | ECC (ECDLP) |
  |---|---|---|---|
  | 80-bit | 3DES | 1024-bit | 160-bit |
  | 128-bit | AES-128 | 3072-bit | 256-bit |
  | 256-bit | AES-256 | 15360-bit | 512-bit |

- **Shor's algorithm (quantum):** O((log n)³) quantum operations. A large-scale quantum computer (~4000 logical qubits for secp256k1) would solve ECDLP in hours. Post-quantum alternatives: CRYSTALS-Kyber (key encapsulation, lattice-based), CRYSTALS-Dilithium (signatures, lattice-based), SPHINCS+ (signatures, hash-based). NIST finalized these in FIPS 203-205 (2024).

---

## 2. Python Implementation: Elliptic Curve from Scratch

```python
from dataclasses import dataclass
from typing import Optional
import secrets
import hashlib
import hmac

# ─────────────────────────────────────────────
# Core Elliptic Curve Arithmetic
# ─────────────────────────────────────────────

@dataclass(frozen=True)
class Point:
    x: Optional[int]
    y: Optional[int]
    
    @property
    def is_infinity(self) -> bool:
        return self.x is None and self.y is None
    
    def __repr__(self) -> str:
        if self.is_infinity:
            return "Point(∞)"
        return f"Point(x={self.x:#066x},\n      y={self.y:#066x})"


class EllipticCurve:
    """
    Elliptic curve E: y² = x³ + ax + b over F_p (prime field).
    Implements complete point arithmetic using only Python integers.
    """
    def __init__(self, a: int, b: int, p: int):
        self.a = a
        self.b = b
        self.p = p
        self.INF = Point(None, None)  # Point at infinity (identity element)
        assert (4 * a**3 + 27 * b**2) % p != 0, "Curve is singular"
    
    def modinv(self, a: int, p: int) -> int:
        """
        Modular inverse via Fermat's little theorem: a^(p-2) mod p.
        Requires p prime and a ≢ 0 (mod p).
        """
        if a % p == 0:
            raise ValueError("No inverse: a ≡ 0 (mod p)")
        return pow(a, p - 2, p)
    
    def is_on_curve(self, P: Point) -> bool:
        if P.is_infinity:
            return True
        lhs = pow(P.y, 2, self.p)
        rhs = (pow(P.x, 3, self.p) + self.a * P.x + self.b) % self.p
        return lhs == rhs
    
    def neg(self, P: Point) -> Point:
        """Return -P = (x, -y mod p)."""
        if P.is_infinity:
            return P
        return Point(P.x, (-P.y) % self.p)
    
    def add(self, P: Point, Q: Point) -> Point:
        """
        Complete point addition: handles all cases including
        P=O, Q=O, P=Q (doubling), P=-Q (returns O).
        """
        if P.is_infinity:
            return Q
        if Q.is_infinity:
            return P
        
        if P.x == Q.x:
            if P.y != Q.y:
                # P = -Q, so P + Q = O
                return self.INF
            elif P.y == 0:
                # Tangent is vertical: 2P = O
                return self.INF
            else:
                # P = Q: use doubling formula
                return self.double(P)
        
        # Standard addition: P ≠ Q, x_P ≠ x_Q
        lam = (Q.y - P.y) * self.modinv(Q.x - P.x, self.p) % self.p
        x_r = (lam * lam - P.x - Q.x) % self.p
        y_r = (lam * (P.x - x_r) - P.y) % self.p
        return Point(x_r, y_r)
    
    def double(self, P: Point) -> Point:
        """Point doubling using tangent slope."""
        if P.is_infinity or P.y == 0:
            return self.INF
        
        lam = (3 * P.x * P.x + self.a) * self.modinv(2 * P.y, self.p) % self.p
        x_r = (lam * lam - 2 * P.x) % self.p
        y_r = (lam * (P.x - x_r) - P.y) % self.p
        return Point(x_r, y_r)
    
    def scalar_mul(self, k: int, P: Point) -> Point:
        """
        Double-and-add scalar multiplication.
        Computes k·P in O(log k) point operations.

        WARNING: Not constant-time. The loop branches on key bits, making this
        vulnerable to timing side-channels. Production code uses the Montgomery
        ladder (see Section 8.1) which performs the same number of operations
        regardless of the secret key value.
        """
        if k < 0:
            return self.scalar_mul(-k, self.neg(P))
        
        R = self.INF
        addend = P
        
        while k:
            if k & 1:       # if LSB is set
                R = self.add(R, addend)
            addend = self.double(addend)
            k >>= 1
        
        return R


# ─────────────────────────────────────────────
# secp256k1 Parameters (Bitcoin's Curve)
# ─────────────────────────────────────────────

# p = 2^256 - 2^32 - 977 (prime)
_p = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F

# Group order n (number of points on the curve)
_n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141

# Generator point G
_Gx = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798
_Gy = 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8

secp256k1 = EllipticCurve(a=0, b=7, p=_p)
G = Point(_Gx, _Gy)

def verify_generator():
    """Verify that n·G = O (G has order n)."""
    assert secp256k1.is_on_curve(G), "G is not on secp256k1!"
    O = secp256k1.scalar_mul(_n, G)
    assert O.is_infinity, f"n·G should be O but got {O}"
    # Also verify 1·G = G
    assert secp256k1.scalar_mul(1, G) == G
    # And 2·G makes sense
    G2 = secp256k1.double(G)
    assert secp256k1.is_on_curve(G2)
    print("✓ secp256k1 generator verified: n·G = O")

# ── A note on secp256k1's generator G ──────────────────────────────────────
# G.x = 0x79BE667E...  — how was this chosen?
#
# secp256k1 is defined by the Standards for Efficient Cryptography Group (SECG).
# Given curve parameters (a=0, b=7, p), G is the lexicographically smallest point
# with the required group order n. The SECG standard (SEC 2 v2.0) documents the
# derivation. The specific value G.x was not backdoored: given p and n, finding
# a "backdoored" G would require solving ECDLP (circular), or implanting a
# trapdoor in n itself — but n is proven prime and matches Schoof/SEA algorithm
# output on (a, b, p). Independent verification: Python's sympy can check
# primality of n, and the Weil bound guarantees |#E(F_p) - (p+1)| ≤ 2√p.
# For secp256k1: n ≈ p (they differ by only ~4.3 billion), consistent with the
# trace of Frobenius t ≈ 4.3×10⁹ << 2√p ≈ 2^128.  ✓

# verify_generator()  # uncomment to run (takes a few seconds)
```

**Projective coordinates (production optimization):** Our implementation calls `modinv()` in every `add()` and `double()` — each `modinv()` costs ~256 squarings via Fermat's little theorem. Production libraries use **Jacobian projective coordinates**, representing (X, Y, Z) where the affine point is (X/Z², Y/Z³). Projective point addition requires only field multiplications — no division! The inverse is deferred to the end: one `modinv()` for the entire scalar multiplication, converting the final (X, Y, Z) back to affine (X/Z², Y/Z³). This gives ~10x speedup.

**Key insight on efficiency:** The inner loop runs `len(bin(k)) - 2` ≈ 256 times for 256-bit k. Each iteration does one doubling; on average half the iterations also do an addition. Total: ~256 doublings + ~128 additions. Each doubling/addition costs 1 modular inverse (a `pow(a, p-2, p)` call ≈ 256 multiplications) + a few multiplications. Grand total: ~50,000 multiplications per scalar multiplication — feasible in milliseconds on modern hardware.

---

## 3. ECDH Key Exchange

### 3.1 Protocol Description

**Setup:** Both parties agree on curve E, generator G, and group order n. These are public.

**Key generation:**
```
Alice: a ←$  Z_n*    (random private key)
       A = a·G        (public key)

Bob:   b ←$  Z_n*
       B = b·G
```

**Key exchange:**
```
Alice sends A → Bob
Bob sends B → Alice

Alice computes: S = a·B = a·(b·G) = (ab)·G
Bob computes:   S = b·A = b·(a·G) = (ab)·G
```

Both arrive at the same point S. They use S.x (the x-coordinate) as the shared secret, then derive a symmetric key via HKDF.

**Security:** An eavesdropper sees G, A=a·G, B=b·G. Computing (ab)·G from (a·G) and (b·G) without knowing a or b is the **Computational Diffie-Hellman (CDH) problem**, which is at least as hard as ECDLP. No polynomial-time classical algorithm is known.

### 3.2 Python Implementation

```python
# ─────────────────────────────────────────────
# ECDH on secp256k1
# ─────────────────────────────────────────────

def ecdh_keygen(curve: EllipticCurve, G: Point, n: int) -> tuple[int, Point]:
    """Generate an ECDH keypair (private_key, public_key)."""
    private = secrets.randbelow(n - 1) + 1   # uniform in [1, n-1]
    public  = curve.scalar_mul(private, G)
    return private, public


def ecdh_shared_secret(curve: EllipticCurve, private_key: int, other_public: Point) -> int:
    """
    Compute ECDH shared secret.
    Returns x-coordinate of k·Q as integer.
    
    NOTE: In production, always validate other_public is on the curve and
    is not the point at infinity before this call.
    """
    if not curve.is_on_curve(other_public):
        raise ValueError("Public key is not on the curve")
    if other_public.is_infinity:
        raise ValueError("Public key is the point at infinity")
    
    shared_point = curve.scalar_mul(private_key, other_public)
    
    if shared_point.is_infinity:
        raise ValueError("Shared point is infinity — possible small subgroup attack")
    
    return shared_point.x


# ─────────────────────────────────────────────
# HKDF (RFC 5869) — derive symmetric key from shared secret
# ─────────────────────────────────────────────

def hkdf_extract(salt: bytes, ikm: bytes) -> bytes:
    """
    HKDF-Extract (RFC 5869 §2.2): HMAC-SHA256(salt, IKM) → pseudorandom key (PRK).

    The salt acts as the HMAC key, and IKM (input key material) is the data.
    This is backwards from intuition but correct per RFC 5869 — salt "randomizes"
    the extraction even when IKM has low entropy (e.g., a short passphrase).
    Without a salt, a zero-filled salt of hash length is used.
    """
    if not salt:
        salt = bytes(32)  # 32 zero bytes (SHA-256 hash length)
    return hmac.new(salt, ikm, hashlib.sha256).digest()


def hkdf_expand(prk: bytes, info: bytes, length: int) -> bytes:
    """
    HKDF-Expand: expand pseudorandom key to `length` bytes.
    Uses HMAC-SHA256 in counter mode.
    """
    hash_len = 32  # SHA-256 output
    n = (length + hash_len - 1) // hash_len
    if n > 255:
        raise ValueError("HKDF cannot produce more than 255*HashLen bytes")
    
    okm = b""
    t = b""
    for i in range(1, n + 1):
        t = hmac.new(prk, t + info + bytes([i]), hashlib.sha256).digest()
        okm += t
    
    return okm[:length]


def hkdf(ikm: bytes, length: int, salt: bytes = b"", info: bytes = b"") -> bytes:
    """Full HKDF: extract then expand."""
    prk = hkdf_extract(salt, ikm)
    return hkdf_expand(prk, info, length)


def ecdh_demo():
    """Complete ECDH key exchange demonstration."""
    # Alice
    alice_priv, alice_pub = ecdh_keygen(secp256k1, G, _n)
    
    # Bob
    bob_priv, bob_pub = ecdh_keygen(secp256k1, G, _n)
    
    # Key exchange
    alice_secret = ecdh_shared_secret(secp256k1, alice_priv, bob_pub)
    bob_secret   = ecdh_shared_secret(secp256k1, bob_priv,   alice_pub)
    
    assert alice_secret == bob_secret, "ECDH failed: secrets don't match!"
    
    # Derive 32-byte AES key from shared secret
    raw = alice_secret.to_bytes(32, 'big')
    aes_key = hkdf(ikm=raw, length=32, info=b"ecdh secp256k1 v1")
    
    print(f"✓ ECDH shared secret (first 8 bytes): {raw[:8].hex()}")
    print(f"✓ Derived AES-256 key:                {aes_key.hex()}")


# ecdh_demo()  # uncomment to run
```

### 3.3 X25519 and Why It Matters

**Curve25519** (defined by Bernstein, 2006) uses the Montgomery form:

```
y² = x³ + 486662x² + x   over F_{2^255 - 19}
```

Advantages over secp256k1 for key exchange:

| Property | secp256k1 | Curve25519 (X25519) |
|---|---|---|
| Addition formulas | Incomplete (special cases) | Complete (no special cases) |
| Constant-time impl | Hard (requires care) | Natural (Montgomery ladder) |
| Cofactor h | 1 | 8 |
| Small-subgroup risk | None (h=1) | Mitigated by multiplying by h |
| Standardization | Bitcoin, many TLS | RFC 7748, TLS 1.3 default |

**Cofactor of 8:** Curve25519's group has order 8·ℓ where ℓ is a large prime. The cofactor-8 subgroup contains 8 small-order points. If ECDH doesn't multiply by the cofactor (clamping in X25519 does this implicitly), an attacker can send a small-order point as their public key, causing the shared secret to take only a few values — leaking bits of the private key. X25519 sets k_0 = k_1 = k_2 = 0 (making k divisible by 8) and k_{255} = 0, k_{254} = 1 (preventing small values), neutralizing this attack.

**Montgomery ladder** for X25519:
```
R0, R1 = 1, P
for bit in reversed(k_bits):
    if bit == 0:
        R1 = add(R0, R1);  R0 = double(R0)
    else:
        R0 = add(R0, R1);  R1 = double(R1)
return R0
```
Always executes exactly 2 operations per bit, regardless of key bits — no timing leak.

---

## 4. Digital Signatures: ECDSA and EdDSA

### 4.1 ECDSA — Elliptic Curve Digital Signature Algorithm

**Signing** (private key d, message m):

```
1. k ←$ [1, n-1]          (random or deterministic nonce)
2. R = k·G
3. r = R.x mod n          (if r=0, retry)
4. e = H(m)               (hash of message, truncated to n bits)
5. s = k^{-1}·(e + r·d) mod n    (if s=0, retry)
6. Signature = (r, s)
```

**Verification** (public key Q = d·G, message m, signature (r, s)):

```
1. Check: 1 ≤ r,s ≤ n-1
2. e = H(m)
3. w = s^{-1} mod n
4. u1 = e·w mod n
5. u2 = r·w mod n
6. X = u1·G + u2·Q
7. Accept iff X.x mod n == r
```

**Why does it work?**

```
X = u1·G + u2·Q
  = e·w·G + r·w·(d·G)
  = w·(e + r·d)·G
  = k^{-1}·(e + r·d)^{-1}·(e + r·d)·(k·G)

Wait — let me redo this properly:
  w = s^{-1} = (k^{-1}(e + rd))^{-1} = k(e+rd)^{-1}
  u1·G + u2·Q = ew·G + rw·d·G
              = w(e + rd)·G
              = k(e+rd)^{-1}·(e+rd)·G
              = k·G = R
So X.x = R.x = r  ✓
```

### 4.2 The Sony PS3 Catastrophe (2010)

Sony used ECDSA to sign PlayStation 3 game executables. A security researcher (fail0verflow) discovered that Sony's signing implementation set **k = 1 for every signature**. Since R = 1·G is a constant, r = R.x was identical across all signatures.

**Attack:** Attacker observes two valid signatures (r, s1) and (r, s2) for messages m1 and m2 (same r means same k):

```
s1 = k^{-1}(e1 + r·d) mod n
s2 = k^{-1}(e2 + r·d) mod n

s1 - s2 = k^{-1}(e1 - e2) mod n
k = (e1 - e2)(s1 - s2)^{-1} mod n          ← k recovered!
d = r^{-1}(s1·k - e1) mod n                ← private key recovered!
```

This broke the entire PS3 security model. **All** game signatures could now be forged, enabling homebrew software and piracy.

### 4.3 RFC 6979: Deterministic ECDSA

RFC 6979 eliminates the random k by deriving it deterministically:

```
K = 0x00...00 (32 bytes)
V = 0x01...01 (32 bytes)

K = HMAC-SHA256(K, V || 0x00 || int_to_bytes(d) || H(m))
V = HMAC-SHA256(K, V)
K = HMAC-SHA256(K, V || 0x01 || int_to_bytes(d) || H(m))
V = HMAC-SHA256(K, V)

k = bits2int(HMAC-SHA256(K, V)) mod n
# if k == 0 or k ≥ n, iterate
```

**Properties:**
- Same (d, m) always produces same k — reproducible, testable
- Different d or different m produces unpredictably different k
- No RNG required during signing — no "I forgot to seed the RNG" bugs
- Cryptographically indistinguishable from random k to an observer who doesn't know d

### 4.4 Python ECDSA Implementation

```python
# ─────────────────────────────────────────────
# ECDSA Signature Scheme
# ─────────────────────────────────────────────

class ECDSA:
    def __init__(self, curve: EllipticCurve, G: Point, n: int):
        self.curve = curve
        self.G = G
        self.n = n
    
    def _hash_message(self, message: bytes) -> int:
        """Hash message and truncate to n bits."""
        h = hashlib.sha256(message).digest()
        # Truncate to bit-length of n
        e = int.from_bytes(h, 'big')
        n_bits = self.n.bit_length()
        h_bits = len(h) * 8
        if h_bits > n_bits:
            e >>= (h_bits - n_bits)
        return e
    
    def _rfc6979_k(self, d: int, message: bytes) -> int:
        """
        RFC 6979 deterministic nonce generation.
        Produces unique, unpredictable k from (private_key, message).
        """
        h = hashlib.sha256(message).digest()
        d_bytes = d.to_bytes(32, 'big')
        
        V = b'\x01' * 32
        K = b'\x00' * 32
        
        K = hmac.new(K, V + b'\x00' + d_bytes + h, hashlib.sha256).digest()
        V = hmac.new(K, V, hashlib.sha256).digest()
        K = hmac.new(K, V + b'\x01' + d_bytes + h, hashlib.sha256).digest()
        V = hmac.new(K, V, hashlib.sha256).digest()
        
        while True:
            V = hmac.new(K, V, hashlib.sha256).digest()
            k = int.from_bytes(V, 'big')
            if 1 <= k < self.n:
                return k
            K = hmac.new(K, V + b'\x00', hashlib.sha256).digest()
            V = hmac.new(K, V, hashlib.sha256).digest()
    
    def sign(self, message: bytes, private_key: int,
             deterministic: bool = True) -> tuple[int, int]:
        """
        Sign message with ECDSA.
        Uses RFC 6979 deterministic k by default.
        """
        e = self._hash_message(message)
        
        while True:
            if deterministic:
                k = self._rfc6979_k(private_key, message)
            else:
                k = secrets.randbelow(self.n - 1) + 1
            
            R = self.curve.scalar_mul(k, self.G)
            r = R.x % self.n
            if r == 0:
                continue
            
            # pow(k, -1, n) computes modular inverse using Python 3.8+ built-in
            # (internally uses extended Euclidean algorithm — faster than Fermat)
            s = pow(k, -1, self.n) * (e + r * private_key) % self.n
            if s == 0:
                continue
            
            return (r, s)
    
    def verify(self, message: bytes, signature: tuple[int, int],
               public_key: Point) -> bool:
        """Verify ECDSA signature."""
        r, s = signature
        
        if not (1 <= r < self.n and 1 <= s < self.n):
            return False
        
        if not self.curve.is_on_curve(public_key):
            return False
        
        e = self._hash_message(message)
        w = pow(s, -1, self.n)
        u1 = e * w % self.n
        u2 = r * w % self.n
        
        X = self.curve.add(
            self.curve.scalar_mul(u1, self.G),
            self.curve.scalar_mul(u2, public_key)
        )
        
        if X.is_infinity:
            return False
        
        return X.x % self.n == r


def ecdsa_demo():
    ecdsa = ECDSA(secp256k1, G, _n)
    
    # Generate key pair
    private_key = secrets.randbelow(_n - 1) + 1
    public_key = secp256k1.scalar_mul(private_key, G)
    
    message = b"ECDSA test message for secp256k1"
    sig = ecdsa.sign(message, private_key)
    
    assert ecdsa.verify(message, sig, public_key), "Signature verification failed!"
    
    # Tampered message should fail
    tampered = b"ECDSA test message for secp256k2"
    assert not ecdsa.verify(tampered, sig, public_key), "Should have rejected tampered message!"
    
    print(f"✓ ECDSA sign/verify passed")
    print(f"  r = {sig[0]:#066x}")
    print(f"  s = {sig[1]:#066x}")


# ecdsa_demo()  # uncomment to run
```

### 4.4a Signature Malleability

ECDSA has a subtle property: if (r, s) is a valid signature, so is (r, n-s). This is because the verification equation is symmetric in s and n-s (the second is just the negation of the scalar multiple).

**Bitcoin Bug (CVE-2014, BIP 62):** An attacker could take a valid transaction signature (r, s) and replace s with (n-s) before the transaction was mined. The transaction ID (txid) is computed from the raw transaction bytes including the signature — so the new signature produced a different txid. The transaction was still valid and would spend the same coins, but with a different hash. This allowed exchange operators to claim "I never received txid X" (because txid X was never mined — only the malleated version was), enabling double-spend claims in specific wallet architectures.

**Fix (BIP 66, later enforced by SegWit):** Require `s ≤ n/2`. The sign function normalizes s:
```python
if s > self.n // 2:
    s = self.n - s   # use the "low-s" form
```

This makes signatures canonical — there's only one valid (r, s) per signing event.

### 4.5 EdDSA (Ed25519): The Modern Alternative

Ed25519 uses a **twisted Edwards curve** over F_{2^255-19}:

```
-x² + y² = 1 + d·x²·y²,   d = -121665/121666 mod p
```

Edwards curves have **complete addition formulas** — no special cases for the identity or for P = -Q. This makes constant-time implementation straightforward.

**Key generation:**
```
1. Private key: 32 random bytes b
2. Hash: (h_0, ..., h_{63}) = SHA-512(b)
3. Scalar: a = 2^254 + 8·⌊h_0...h_{31}/8⌋  (clamped)
4. Public key: A = a·B  (B is the base point)
```

**Signing** (private key (a, prefix), message m, public key A):
```
1. r = SHA-512(prefix || m) as integer mod l
2. R = r·B
3. S = (r + SHA-512(R || A || m)·a) mod l
4. Signature = R || S   (64 bytes)
```

**Verification** (public key A, message m, signature (R, S)):
```
Check: 8·S·B == 8·R + 8·SHA-512(R || A || m)·A
```

The factor of 8 (cofactor) prevents small-subgroup attacks during verification.

**Why EdDSA beats ECDSA:**
- No per-signature randomness required (no nonce = no Sony-style bug)
- Complete addition formulas → easier to implement correctly
- Faster (Edwards curves have more efficient addition formulas)
- Built-in cofactor handling
- Shorter signatures (64 bytes vs 71-72 bytes DER-encoded ECDSA)

---

## 5. Zero-Knowledge Proofs: Schnorr Protocol

### 5.1 What Is a Zero-Knowledge Proof?

A ZKP protocol between a **Prover** P and a **Verifier** V allows P to convince V that a statement is true **without revealing why it's true** — specifically, without revealing any secret witness.

Formal requirements:
- **Completeness:** If P knows the secret, an honest execution always convinces V.
- **Soundness:** If P doesn't know the secret, V accepts with only negligible probability (over V's randomness). Quantified: probability ≤ 1/n for group order n.
- **Zero-knowledge:** The transcript (P, V) can be **simulated** without knowing the secret. V learns nothing about the secret beyond the statement being true.

### 5.2 Schnorr Identification Protocol (Interactive)

**Statement:** "I know x such that Y = x·G" (for public Y).

```
Prover (knows x)              Verifier (knows G, Y, n)
──────────────────────────────────────────────────────
r ←$ [1,n-1]
A = r·G
                  A
                ────→

                  c
                ←────          c ←$ [1,n-1]

s = (r + c·x) mod n
                  s
                ────→
                              Check: s·G == A + c·Y  ?
```

**Why does verification work?**
```
s·G = (r + cx)·G = r·G + c·(x·G) = A + c·Y  ✓
```

**Soundness via rewinding argument:** If a cheating prover P* can answer two different challenges c ≠ c' for the same commitment A, then:
```
s·G = A + c·Y   and   s'·G = A + c'·Y
(s - s')·G = (c - c')·Y
Y = (s-s')·(c-c')^{-1}·G
x = (s-s')·(c-c')^{-1} mod n    ← private key extracted!
```

So if P* succeeds twice with different challenges, we can extract x. This means P* must know x — soundness follows.

**Zero-knowledge via simulation:** Given any challenge c, a simulator can produce a valid-looking transcript (A, c, s) without knowing x:
1. Pick s ←$ [1,n-1]
2. Compute A = s·G - c·Y
3. Output (A, c, s)

The simulated transcript is **identically distributed** to a real transcript (both A and s are uniform in their ranges). V cannot distinguish simulation from reality.

### 5.3 Non-Interactive Schnorr (Fiat-Shamir Transform)

Replace the verifier's random challenge with a hash of the public inputs:

```
c = H(G || Y || A)
```

The prover computes A, c, and s themselves. The proof is a pair (A, s).

**Security:** In the **Random Oracle Model** (treating H as a truly random function), this is provably secure. The intuition: if the prover could choose A after seeing c (cheating), they'd need to query the oracle with (G,Y,A) first, which fixes c. But c was fixed before A was chosen — contradiction.

**Schnorr signatures:** Schnorr's scheme is essentially a Fiat-Shamir Schnorr proof applied to a message m:

```
c = H(G || Y || A || m)
```

This binds the proof to message m, producing a digital signature. Ed25519 is a variant of this construction.

### 5.4 Python Implementation

```python
# ─────────────────────────────────────────────
# Schnorr Zero-Knowledge Proof (Non-Interactive, Fiat-Shamir)
# ─────────────────────────────────────────────

class SchnorrProof:
    """
    Non-interactive Schnorr proof of knowledge of discrete logarithm.
    Proves: "I know x such that Y = x·G" without revealing x.
    """
    def __init__(self, curve: EllipticCurve, G: Point, n: int):
        self.curve = curve
        self.G = G
        self.n = n
    
    def _challenge(self, Y: Point, A: Point) -> int:
        """Fiat-Shamir hash: H(G || Y || A) → challenge c."""
        data = (
            self.G.x.to_bytes(32, 'big') +
            self.G.y.to_bytes(32, 'big') +
            Y.x.to_bytes(32, 'big') +
            Y.y.to_bytes(32, 'big') +
            A.x.to_bytes(32, 'big') +
            A.y.to_bytes(32, 'big')
        )
        h = hashlib.sha256(data).digest()
        return int.from_bytes(h, 'big') % self.n
    
    def prove(self, secret: int) -> tuple[Point, Point, int]:
        """
        Generate a Schnorr proof for secret x.
        Returns (Y, A, s) where:
          Y = x·G  (public key / statement)
          A = r·G  (commitment)
          s = r + c·x mod n  (response)
        """
        Y = self.curve.scalar_mul(secret, self.G)
        r = secrets.randbelow(self.n - 1) + 1
        A = self.curve.scalar_mul(r, self.G)
        c = self._challenge(Y, A)
        s = (r + c * secret) % self.n
        return (Y, A, s)
    
    def verify(self, Y: Point, A: Point, s: int) -> bool:
        """
        Verify a Schnorr proof.
        Checks: s·G == A + c·Y
        """
        if not self.curve.is_on_curve(Y):
            return False
        if not self.curve.is_on_curve(A):
            return False
        if not (1 <= s < self.n):
            return False
        
        c = self._challenge(Y, A)
        
        lhs = self.curve.scalar_mul(s, self.G)
        rhs = self.curve.add(A, self.curve.scalar_mul(c, Y))
        
        return lhs == rhs
    
    def simulate(self, Y: Point, c: int) -> tuple[Point, int]:
        """
        Simulator: produce (A, s) without knowing x.
        A = s·G - c·Y for random s.
        This demonstrates zero-knowledge property.
        """
        s = secrets.randbelow(self.n - 1) + 1
        sG  = self.curve.scalar_mul(s, self.G)
        cY  = self.curve.scalar_mul(c, Y)
        neg_cY = self.curve.neg(cY)
        A = self.curve.add(sG, neg_cY)
        return A, s


def schnorr_tests():
    """Comprehensive tests for the Schnorr proof system."""
    schnorr = SchnorrProof(secp256k1, G, _n)
    
    # Test 1: Valid proof verifies
    secret = secrets.randbelow(_n - 1) + 1
    Y, A, s = schnorr.prove(secret)
    assert schnorr.verify(Y, A, s), "Valid proof should verify!"
    print("✓ Test 1 passed: valid proof verifies")
    
    # Test 2: Tampered s fails
    assert not schnorr.verify(Y, A, (s + 1) % _n), "Tampered s should fail!"
    print("✓ Test 2 passed: tampered s rejected")
    
    # Test 3: Wrong public key fails
    wrong_Y = secp256k1.scalar_mul(secret + 1, G)
    assert not schnorr.verify(wrong_Y, A, s), "Wrong Y should fail!"
    print("✓ Test 3 passed: wrong public key rejected")
    
    # Test 4: Tampered commitment A fails
    _, A2, s2 = schnorr.prove(secrets.randbelow(_n - 1) + 1)
    assert not schnorr.verify(Y, A2, s), "Tampered A should fail!"
    print("✓ Test 4 passed: tampered commitment rejected")
    
    # Test 5: Simulator output verifies (ZK property demonstration)
    c_test = secrets.randbelow(_n)
    A_sim, s_sim = schnorr.simulate(Y, c_test)
    # The simulated (A_sim, s_sim) should satisfy s·G = A + c·Y for the given c
    lhs = secp256k1.scalar_mul(s_sim, G)
    rhs = secp256k1.add(A_sim, secp256k1.scalar_mul(c_test, Y))
    assert lhs == rhs, "Simulator should produce valid transcripts!"
    print("✓ Test 5 passed: simulator produces valid-looking transcripts (ZK property)")
    
    print("\n✓ All Schnorr tests passed!")


# schnorr_tests()  # uncomment to run
```

### 5.5 Schnorr Aggregate Signatures (MuSig2)

One of Schnorr's profound advantages over ECDSA is **linearity**: multiple parties can collaboratively produce a single signature that's indistinguishable from a single-signer Schnorr signature.

**Naive aggregation attempt (broken):**
- Each of n signers has key (x_i, Y_i = x_i·G)
- Aggregate public key: Y = Y_1 + Y_2 + ... + Y_n
- Each signs with nonce r_i, commitment A_i = r_i·G, response s_i = r_i + c·x_i
- Aggregate: A = ΣA_i, s = Σs_i
- Verification: s·G = A + c·Y  ✓ (by linearity of scalar mult)

**The rogue key attack:** Malicious party n+1 announces Y_{n+1} = Y_target - (Y_1 + ... + Y_n). Then the "aggregate" key is Y_target — any key the attacker wants. They can produce valid signatures unilaterally.

**MuSig2 (Maxwell et al., 2021) fixes this:**
1. Key aggregation uses **key coefficient** a_i = H(L || Y_i) where L = {Y_1,...,Y_n} set. Aggregate key: Ỹ = Σ(a_i · Y_i). This prevents rogue key because knowing L is required to compute a_i.
2. Two-round nonce commitment: each signer commits to multiple nonces (R_{i,1}, R_{i,2}) before revealing them. This prevents a Wagner's algorithm attack on the nonce.

**Result:** An n-of-n multisignature that verifies as a single Schnorr signature in O(1) time, regardless of n. Bitcoin Taproot (BIP 340) uses Schnorr for exactly this reason.

```python
# MuSig2 key aggregation (simplified, without full MuSig2 nonce protocol)
def musig2_key_aggregate(curve, G, n_order, public_keys: list[Point]) -> Point:
    """
    Aggregate n public keys into a single key.
    Uses key coefficients to prevent rogue-key attacks.
    """
    # L = hash of all public keys concatenated
    L_data = b"".join(
        pk.x.to_bytes(32, 'big') + pk.y.to_bytes(32, 'big')
        for pk in public_keys
    )
    L_hash = hashlib.sha256(b"KeyAgg list" + L_data).digest()
    
    agg_key = curve.INF
    for pk in public_keys:
        # Key coefficient: a_i = H("KeyAgg coefficient" || L || Y_i)
        pk_bytes = pk.x.to_bytes(32, 'big') + pk.y.to_bytes(32, 'big')
        coeff_data = b"KeyAgg coefficient" + L_hash + pk_bytes
        a_i = int.from_bytes(hashlib.sha256(coeff_data).digest(), 'big') % n_order
        # Aggregate: Ỹ += a_i · Y_i
        agg_key = curve.add(agg_key, curve.scalar_mul(a_i, pk))
    
    return agg_key
```

**Practical impact:** A 2-of-2 Bitcoin multisig historically required 2 signatures and 2 public keys on-chain (~220 bytes). With MuSig2 + Taproot, it's indistinguishable from a single signature (64 bytes) — 72% space savings and identical verification cost.

---

## 6. TLS 1.3 Handshake — Deep Trace

### 6.1 What Changed From TLS 1.2

TLS 1.2 had critical problems:
- **RSA key exchange** had no forward secrecy: compromise of server's RSA private key decrypts all past sessions.
- **2 RTT** for full handshake (slower).
- **Renegotiation** was complex and introduced vulnerabilities (CRIME, BEAST, POODLE all exploited TLS 1.2 weaknesses).

TLS 1.3 (RFC 8446) eliminates RSA key exchange entirely, mandates ephemeral Diffie-Hellman, and collapses the handshake to **1 RTT** (0-RTT for resumed sessions).

### 6.2 Step-by-Step Handshake

**Step 1 — ClientHello**
```
ClientHello {
  legacy_version: 0x0303  (TLS 1.2 for middlebox compat)
  random: 32 bytes (client_random)
  legacy_session_id: 32 bytes (for middlebox compat)
  cipher_suites: [
    TLS_AES_128_GCM_SHA256,
    TLS_AES_256_GCM_SHA384,
    TLS_CHACHA20_POLY1305_SHA256
  ]
  extensions: [
    supported_versions: [TLS 1.3]
    key_share: {
      secp256r1: client_ecdh_public_key,   # 65 bytes
      x25519:    client_x25519_public_key  # 32 bytes
    }
    signature_algorithms: [ecdsa_secp256r1_sha256, ...]
    server_name: "example.com"  (SNI)
  ]
}
```

The client sends its ephemeral DH key share speculatively (guessing the server's preferred group). If the guess is wrong, the server sends a HelloRetryRequest (rare).

**Step 2 — ServerHello**
```
ServerHello {
  random: 32 bytes (server_random)
  cipher_suite: TLS_AES_256_GCM_SHA384
  extensions: [
    supported_versions: TLS 1.3
    key_share: { x25519: server_x25519_public_key }
  ]
}
```

After ServerHello, both sides compute:
```
shared_secret = X25519(server_private, client_x25519_public)
             = X25519(client_private, server_x25519_public)
```

From this point, **all traffic is encrypted**.

**Step 3 — Key Schedule (HKDF-based)**

TLS 1.3's key schedule is a carefully structured HKDF derivation:

```python
# Pseudocode for TLS 1.3 key schedule

def derive_secret(secret, label, messages_hash):
    """HKDF-Expand-Label(secret, label, hash(messages), hash_length)"""
    hkdf_label = length_prefix(b"tls13 " + label) + length_prefix(messages_hash)
    return hkdf_expand(secret, hkdf_label, hash_length)

def hkdf_extract(salt, ikm):
    return hmac.new(salt, ikm, hash_algo).digest()

# Phase 1: Early Secret (for 0-RTT)
early_secret = hkdf_extract(salt=b'\x00'*48, ikm=psk_or_zeros)
derived_early = derive_secret(early_secret, b"derived", sha384(b""))

# Phase 2: Handshake Secret
handshake_secret = hkdf_extract(salt=derived_early, ikm=shared_secret)

client_handshake_traffic = derive_secret(handshake_secret, b"c hs traffic", sha384(hello_messages))
server_handshake_traffic = derive_secret(handshake_secret, b"s hs traffic", sha384(hello_messages))

# Derive actual keys and IVs from traffic secrets
client_write_key = hkdf_expand_label(client_handshake_traffic, b"key", b"", 32)
client_write_iv  = hkdf_expand_label(client_handshake_traffic, b"iv",  b"", 12)
# (similarly for server)

derived_handshake = derive_secret(handshake_secret, b"derived", sha384(b""))

# Phase 3: Master Secret
master_secret = hkdf_extract(salt=derived_handshake, ikm=b'\x00'*48)

client_application_traffic = derive_secret(master_secret, b"c ap traffic", sha384(all_handshake))
server_application_traffic = derive_secret(master_secret, b"s ap traffic", sha384(all_handshake))
```

The transcript hash is taken over **all** handshake messages so far, binding each key to the exact messages exchanged. This prevents cut-and-paste attacks.

**Step 4 — EncryptedExtensions + Certificate + CertificateVerify + Finished**

All sent by server, encrypted under `server_handshake_traffic` keys:
```
EncryptedExtensions: {ALPN, server_name confirmation, ...}
Certificate:         {X.509 certificate chain}
CertificateVerify:   {ECDSA/RSA-PSS signature over SHA-256(transcript)}
Finished:            {HMAC(server_handshake_traffic, transcript)}
```

**CertificateVerify** proves the server possesses the private key matching the certificate — it signs the entire transcript, binding the certificate to this specific session.

**Finished** is an HMAC authenticating all handshake messages. If any byte was tampered with, Finished fails.

**Step 5 — Client Finished**

Client sends its own Finished, then application data flows under `application_traffic` keys.

### 6.3 Forward Secrecy

**Definition:** Compromise of long-term keys (server's certificate private key) does not enable decryption of past recorded sessions.

**How TLS 1.3 achieves it:**

The `shared_secret` is derived from **ephemeral** ECDH keys generated fresh for each handshake. After the handshake completes, both sides delete the ephemeral private keys. The certificate private key is used only in `CertificateVerify` — it signs the transcript but is **not used in key derivation**.

Without the ephemeral private keys, `shared_secret` cannot be recomputed, even with the certificate private key. Forward secrecy is established **at the point where ephemeral keys are deleted**, which is immediately after the handshake.

**0-RTT data:** The first flight of application data uses the PSK (Pre-Shared Key) from a previous session, not ephemeral ECDH. This breaks forward secrecy for 0-RTT data: if the PSK is compromised, 0-RTT data can be decrypted. Additionally, 0-RTT data can be **replayed** by a network attacker (no anti-replay mechanism in the base TLS 1.3 spec). Servers must make 0-RTT handlers idempotent.

### 6.3a Python Sketch: TLS 1.3 Key Schedule

```python
def tls13_key_schedule(shared_secret: bytes, hello_transcript: bytes,
                        full_transcript: bytes, hash_algo=hashlib.sha384):
    """
    Simplified TLS 1.3 key schedule (RFC 8446 §7.1).
    Returns handshake and application traffic secrets.
    
    shared_secret:    ECDH/X25519 output (32 or 56 bytes)
    hello_transcript: SHA-384 of ClientHello + ServerHello
    full_transcript:  SHA-384 of all handshake messages
    """
    hash_len = hash_algo().digest_size  # 48 for SHA-384

    def hkdf_extract_tls(salt: bytes, ikm: bytes) -> bytes:
        return hmac.new(salt, ikm, hash_algo).digest()

    def hkdf_expand_label(prk: bytes, label: str, context: bytes, length: int) -> bytes:
        """RFC 8446 HKDF-Expand-Label."""
        full_label = b"tls13 " + label.encode()
        # HkdfLabel struct: uint16 length, opaque label<7..255>, opaque context<0..255>
        hkdf_label = (
            length.to_bytes(2, 'big') +
            bytes([len(full_label)]) + full_label +
            bytes([len(context)]) + context
        )
        return hkdf_expand(prk, hkdf_label, length)

    def derive_secret(secret: bytes, label: str, transcript: bytes) -> bytes:
        """Derive-Secret(secret, label, messages) = HKDF-Expand-Label(secret, label, Hash(messages), hash_len)"""
        transcript_hash = hash_algo(transcript).digest()
        return hkdf_expand_label(secret, label, transcript_hash, hash_len)

    zeros = bytes(hash_len)

    # Phase 1: Early Secret (PSK = 0 for non-PSK handshake)
    early_secret = hkdf_extract_tls(salt=zeros, ikm=zeros)
    derived = derive_secret(early_secret, "derived", b"")

    # Phase 2: Handshake Secret
    handshake_secret = hkdf_extract_tls(salt=derived, ikm=shared_secret)
    client_hs_traffic = derive_secret(handshake_secret, "c hs traffic", hello_transcript)
    server_hs_traffic = derive_secret(handshake_secret, "s hs traffic", hello_transcript)
    derived2 = derive_secret(handshake_secret, "derived", b"")

    # Phase 3: Master Secret
    master_secret = hkdf_extract_tls(salt=derived2, ikm=zeros)
    client_ap_traffic = derive_secret(master_secret, "c ap traffic", full_transcript)
    server_ap_traffic = derive_secret(master_secret, "s ap traffic", full_transcript)

    # Derive actual write keys and IVs (AES-256-GCM: 32-byte key, 12-byte IV)
    def traffic_keys(traffic_secret: bytes) -> dict:
        return {
            "key": hkdf_expand_label(traffic_secret, "key", b"", 32),
            "iv":  hkdf_expand_label(traffic_secret, "iv",  b"", 12),
        }

    return {
        "client_handshake": traffic_keys(client_hs_traffic),
        "server_handshake": traffic_keys(server_hs_traffic),
        "client_application": traffic_keys(client_ap_traffic),
        "server_application": traffic_keys(server_ap_traffic),
    }
```

This can be validated against Wireshark's TLS 1.3 dissector — capture a TLS 1.3 session with SSLKEYLOGFILE, observe the exported secrets file, and verify that the key schedule above produces matching keys.

### 6.4 Cipher Suite Breakdown

`TLS_AES_256_GCM_SHA384`:
- **AES-256-GCM:** Authenticated Encryption with Associated Data (AEAD). 256-bit key, Galois/Counter Mode provides both confidentiality (CTR encryption) and integrity (GHASH MAC). 12-byte nonce, 16-byte authentication tag.
- **SHA-384:** Used in HKDF for all key derivation. 384-bit security against collision attacks (SHA-256 gives only 128-bit collision resistance).

`TLS_CHACHA20_POLY1305_SHA256`:
- **ChaCha20-Poly1305:** Software-optimized AEAD. Used on mobile devices without AES hardware acceleration (AES-NI). ChaCha20 is a stream cipher; Poly1305 is a one-time MAC. Constant-time on all platforms.

---

## 7. SHA-256 Internals: The Compression Function

### 7.1 Merkle-Damgård Construction

SHA-256 uses the **Merkle-Damgård** structure:

```
Message M
  ↓ padding (append 1-bit, zeros, 64-bit length)
  ↓ split into 512-bit blocks B_0, B_1, ..., B_{L-1}
  
H_0 = IV  (initial hash value, constant)
H_1 = f(H_0, B_0)
H_2 = f(H_1, B_1)
...
H_L = f(H_{L-1}, B_{L-1})

Output = H_L  (256 bits)
```

The compression function `f` takes a 256-bit state and a 512-bit block, outputs a 256-bit state. **Length extension attacks** arise because `H_L` can be used as an initial state for more blocks, without knowing M.

### 7.2 Constants: "Nothing Up My Sleeve"

**Initial hash values H_0..H_7:** Fractional parts of square roots of first 8 primes:
```
H_0 = ⌊frac(√2)  · 2^32⌋ = 0x6a09e667
H_1 = ⌊frac(√3)  · 2^32⌋ = 0xbb67ae85
H_2 = ⌊frac(√5)  · 2^32⌋ = 0x3c6ef372
H_3 = ⌊frac(√7)  · 2^32⌋ = 0xa54ff53a
H_4 = ⌊frac(√11) · 2^32⌋ = 0x510e527f
H_5 = ⌊frac(√13) · 2^32⌋ = 0x9b05688c
H_6 = ⌊frac(√17) · 2^32⌋ = 0x1f83d9ab
H_7 = ⌊frac(√19) · 2^32⌋ = 0x5be0cd19
```

**Round constants K_0..K_{63}:** Fractional parts of cube roots of first 64 primes. Since these are determined by the sequence of primes (publicly known, universally agreed upon), there's no room to backdoor them — any backdoor would require finding a prime sequence that produces specific constants.

### 7.3 Message Schedule and Round Function

For each 512-bit block:
1. **Message schedule:** Expand 16 32-bit words into 64:
   ```
   W_i = M_i                                           for i = 0..15
   W_i = σ1(W_{i-2}) + W_{i-7} + σ0(W_{i-15}) + W_{i-16}   for i = 16..63
   
   σ0(x) = ROTR(x,7)  ⊕ ROTR(x,18) ⊕ SHR(x,3)
   σ1(x) = ROTR(x,17) ⊕ ROTR(x,19) ⊕ SHR(x,10)
   ```

2. **64 rounds:** Each round mixes one word W_i into the 8-word state (a,b,c,d,e,f,g,h):
   ```
   T1 = h + Σ1(e) + Ch(e,f,g) + K_i + W_i
   T2 = Σ0(a) + Maj(a,b,c)
   
   h=g, g=f, f=e, e=d+T1, d=c, c=b, b=a, a=T1+T2
   
   Σ0(x) = ROTR(x,2)  ⊕ ROTR(x,13) ⊕ ROTR(x,22)
   Σ1(x) = ROTR(x,6)  ⊕ ROTR(x,11) ⊕ ROTR(x,25)
   Ch(x,y,z) = (x AND y) XOR (NOT x AND z)   "choice"
   Maj(x,y,z) = (x AND y) XOR (x AND z) XOR (y AND z)  "majority"
   ```

3. **Feedforward (Davies-Meyer construction):**
   ```
   H_i = H_{i-1} + (a,b,c,d,e,f,g,h)  (component-wise mod 2^32)
   ```
   This makes the compression function one-way: even knowing the output state, inverting f would require finding a preimage of the block cipher in a chosen-plaintext scenario, which is assumed hard.

### 7.3a Tracing Round 0 of SHA-256 on Input "abc"

The message `b"abc"` = `0x61 0x62 0x63`. After padding, the 512-bit block is:

```
6162 6380 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0018
```

The first 16 32-bit words W[0..15]:
```
W[0]  = 0x61626380   W[1]  = 0x00000000   ...   W[14] = 0x00000000
W[15] = 0x00000018   (length in bits: 3 × 8 = 24 = 0x18)
```

Round 0, working variables initialized to H (IV):
```
a = 0x6a09e667   b = 0xbb67ae85   c = 0x3c6ef372   d = 0xa54ff53a
e = 0x510e527f   f = 0x9b05688c   g = 0x1f83d9ab   h = 0x5be0cd19
```

Computing round 0:
```python
Sigma1(e=0x510e527f) = ROTR(e,6) ^ ROTR(e,11) ^ ROTR(e,25)
                     = 0x941f4c09 ^ 0x3fd027a0 ^ 0x28773f48
                     = 0xb9f87b91   (XOR chain)

Ch(e, f, g) = (e & f) ^ (~e & g)
            = (0x510e527f & 0x9b05688c) ^ (~0x510e527f & 0x1f83d9ab)
            = 0x11046288 ^ 0x0e818129
            = 0x1f85e3a9

T1 = h + Sigma1(e) + Ch(e,f,g) + K[0] + W[0]
   = 0x5be0cd19 + 0xb9f87b91 + 0x1f85e3a9 + 0x428a2f98 + 0x61626380
   = 0xd9d80a2b   (mod 2^32, with carries discarded)

Sigma0(a=0x6a09e667) = ROTR(a,2) ^ ROTR(a,13) ^ ROTR(a,22)
                     = 0x9a827db9 ^ 0xb3340ff3 ^ 0x79a01a1b
                     = 0xc0f66841   (verify with Python: rotr(a,2)^rotr(a,13)^rotr(a,22))

Maj(a,b,c) = (a&b) ^ (a&c) ^ (b&c)
           = (0x6a09e667 & 0xbb67ae85) ^ (0x6a09e667 & 0x3c6ef372) ^ (0xbb67ae85 & 0x3c6ef372)
           = 0x2a01a605 ^ 0x28086262 ^ 0x38662200
           = 0x1aef6467   (XOR of three terms)

T2 = Sigma0(a) + Maj(a,b,c) = 0xc0f66841 + 0x1aef6467 = 0xdbe5ccA8  (mod 2^32)

New state:
h = 0x1f83d9ab   g = 0x9b05688c   f = 0x510e527f
e = d + T1 = 0xa54ff53a + 0xd9d80a2b = 0x7f27ff65  (mod 2^32)
d = 0x3c6ef372   c = 0xbb67ae85   b = 0x6a09e667
a = T1 + T2 = 0xd9d80a2b + 0xdbe5ccA8 = 0xb5bdd6d3  (mod 2^32)
```

After all 64 rounds plus feedforward, `SHA-256("abc")` =
`0xba7816bf 8f01cfea 414140de 5dae2ec7 3b338c2b a3e8fe9c bac89eb1`.

### 7.4 Python SHA-256 from Scratch

```python
# ─────────────────────────────────────────────
# SHA-256 from Scratch (no crypto libraries)
# ─────────────────────────────────────────────

def sha256_scratch(message: bytes) -> bytes:
    """
    SHA-256 implementation from scratch.
    Implements FIPS 180-4 specification exactly.
    """
    # Round constants: cube roots of first 64 primes
    K = [
        0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5,
        0x3956c25b, 0x59f111f1, 0x923f82a4, 0xab1c5ed5,
        0xd807aa98, 0x12835b01, 0x243185be, 0x550c7dc3,
        0x72be5d74, 0x80deb1fe, 0x9bdc06a7, 0xc19bf174,
        0xe49b69c1, 0xefbe4786, 0x0fc19dc6, 0x240ca1cc,
        0x2de92c6f, 0x4a7484aa, 0x5cb0a9dc, 0x76f988da,
        0x983e5152, 0xa831c66d, 0xb00327c8, 0xbf597fc7,
        0xc6e00bf3, 0xd5a79147, 0x06ca6351, 0x14292967,
        0x27b70a85, 0x2e1b2138, 0x4d2c6dfc, 0x53380d13,
        0x650a7354, 0x766a0abb, 0x81c2c92e, 0x92722c85,
        0xa2bfe8a1, 0xa81a664b, 0xc24b8b70, 0xc76c51a3,
        0xd192e819, 0xd6990624, 0xf40e3585, 0x106aa070,
        0x19a4c116, 0x1e376c08, 0x2748774c, 0x34b0bcb5,
        0x391c0cb3, 0x4ed8aa4a, 0x5b9cca4f, 0x682e6ff3,
        0x748f82ee, 0x78a5636f, 0x84c87814, 0x8cc70208,
        0x90befffa, 0xa4506ceb, 0xbef9a3f7, 0xc67178f2,
    ]
    
    # Initial hash values: square roots of first 8 primes
    H = [
        0x6a09e667, 0xbb67ae85, 0x3c6ef372, 0xa54ff53a,
        0x510e527f, 0x9b05688c, 0x1f83d9ab, 0x5be0cd19,
    ]
    
    M32 = 0xFFFFFFFF  # 32-bit mask
    
    def rotr(x: int, n: int) -> int:
        return ((x >> n) | (x << (32 - n))) & M32
    
    def sigma0(x: int) -> int:  # for message schedule
        return rotr(x, 7) ^ rotr(x, 18) ^ (x >> 3)
    
    def sigma1(x: int) -> int:  # for message schedule
        return rotr(x, 17) ^ rotr(x, 19) ^ (x >> 10)
    
    def Sigma0(x: int) -> int:  # for compression
        return rotr(x, 2) ^ rotr(x, 13) ^ rotr(x, 22)
    
    def Sigma1(x: int) -> int:  # for compression
        return rotr(x, 6) ^ rotr(x, 11) ^ rotr(x, 25)
    
    def Ch(x: int, y: int, z: int) -> int:
        return ((x & y) ^ (~x & z)) & M32   # & binds tighter than ^; mask whole result
    
    def Maj(x: int, y: int, z: int) -> int:
        return (x & y) ^ (x & z) ^ (y & z)
    
    # ── Padding ──────────────────────────────────
    # Append bit '1' (0x80), then zeros, then 64-bit big-endian length.
    # Total padded length must be ≡ 0 (mod 512 bits = 64 bytes).
    bit_len = len(message) * 8
    message += b'\x80'
    while len(message) % 64 != 56:
        message += b'\x00'
    message += bit_len.to_bytes(8, 'big')
    
    assert len(message) % 64 == 0
    
    # ── Process Each 512-bit Block ───────────────
    for block_start in range(0, len(message), 64):
        block = message[block_start:block_start + 64]
        
        # Parse into 16 32-bit words
        W = list(int.from_bytes(block[i:i+4], 'big') for i in range(0, 64, 4))
        
        # Extend to 64 words
        for i in range(16, 64):
            w = sigma1(W[i-2]) + W[i-7] + sigma0(W[i-15]) + W[i-16]
            W.append(w & M32)
        
        # Initialize working variables
        a, b, c, d, e, f, g, h = H
        
        # 64 compression rounds
        for i in range(64):
            T1 = (h + Sigma1(e) + Ch(e, f, g) + K[i] + W[i]) & M32
            T2 = (Sigma0(a) + Maj(a, b, c)) & M32
            h = g
            g = f
            f = e
            e = (d + T1) & M32
            d = c
            c = b
            b = a
            a = (T1 + T2) & M32
        
        # Feedforward (Davies-Meyer): add compressed to previous hash
        H[0] = (H[0] + a) & M32
        H[1] = (H[1] + b) & M32
        H[2] = (H[2] + c) & M32
        H[3] = (H[3] + d) & M32
        H[4] = (H[4] + e) & M32
        H[5] = (H[5] + f) & M32
        H[6] = (H[6] + g) & M32
        H[7] = (H[7] + h) & M32
    
    # Produce final 256-bit digest
    return b''.join(h_i.to_bytes(4, 'big') for h_i in H)


def sha256_test_vectors():
    """Verify our SHA-256 against hashlib on standard test vectors."""
    import hashlib
    
    test_vectors = [
        b"",
        b"abc",
        b"abcdbcdecdefdefgefghfghighijhijkijkljklmklmnlmnomnopnopq",
        b"The quick brown fox jumps over the lazy dog",
        b"a" * 1000,
        b"\x00" * 64,    # exactly one block
        b"\xff" * 128,   # exactly two blocks
    ]
    
    all_pass = True
    for msg in test_vectors:
        expected = hashlib.sha256(msg).digest()
        got      = sha256_scratch(msg)
        status = "✓" if got == expected else "✗"
        if got != expected:
            all_pass = False
        print(f"{status} SHA-256({msg[:20]}{'...' if len(msg)>20 else ''!r}) = {got[:8].hex()}...")
    
    if all_pass:
        print("\n✓ All SHA-256 test vectors passed!")
    else:
        print("\n✗ Some tests FAILED!")


# sha256_test_vectors()  # uncomment to run
```

---

## 8. What Textbooks Don't Tell You

### 8.1 Timing Attacks and Constant-Time Code

Cryptographic implementations **must not branch on secret data**. Python's `pow(a, p-2, p)` uses the built-in square-and-multiply algorithm which, in CPython, branches on key bits via the code path taken in integer arithmetic. On modern CPUs, these branches cause measurable timing differences (Kocher's 1996 paper demonstrated RSA key extraction via timing).

**Practical example:** A naive RSA decryption that's 0.001% slower for a particular key bit pattern allows recovery of the private key with ~1 million oracle queries.

**Solutions:**
- **Montgomery ladder:** For scalar multiplication, always perform both doubling and addition regardless of key bit. The key insight: branch on `bit` only determines *which register* stores which value — not whether arithmetic happens:
  ```python
  def scalar_mul_montgomery_ladder(self, k: int, P: Point) -> Point:
      """
      Constant-time scalar multiplication via Montgomery ladder.
      Executes exactly 1 doubling + 1 addition per key bit,
      regardless of bit value — no timing side-channel.
      """
      R0 = self.INF   # accumulates k·P
      R1 = P          # always one step ahead: (k+1)·P
      
      for bit in reversed(range(k.bit_length())):
          if (k >> bit) & 1:
              R0 = self.add(R0, R1)    # both ops happen every iteration
              R1 = self.double(R1)
          else:
              R1 = self.add(R0, R1)    # branch only swaps which is which
              R0 = self.double(R0)
      
      return R0
      # NOTE: Even this Python version leaks timing via branch prediction.
      # True constant-time requires conditional swap using bitmasking in C/Rust.
  ```
  
- **Blinding:** Multiply the scalar k by a random multiple of the group order before scalar multiplication, then strip it: `k' = k + random·n`. The operation takes a path that depends on k' (random), not k.

- **Constant-time comparison:** Replace `a == b` with `hmac.compare_digest(a, b)` (uses `memcmp`-style loop) to prevent timing leaks in authentication checks.

### 8.2 The Random Oracle Model

Many cryptographic security proofs assume the hash function H is a **random oracle** — a publicly accessible truly random function mapping arbitrary strings to fixed-length outputs, with no structure an attacker can exploit.

Real hash functions (SHA-256, SHA-3) are **not** random oracles:
- They're deterministic, finite, algebraically structured
- Theoretical "correlation" attacks exist in idealized settings
- Multi-collision attacks (finding 2^k messages with the same hash) take 2^{n/2+1} queries for Merkle-Damgård (better than for a RO)

**Why we still use the ROM:** No practical attacks against ROM-based proofs are known when using SHA-256 or SHA-3 as the hash function. The ROM provides a structured way to make security arguments for complex protocols. The alternative — standard model proofs — often require much stronger computational assumptions or produce less efficient constructions.

### 8.3 Cofactor Attacks and Small-Subgroup Danger

When curve cofactor h > 1, there exist **small-order points** of order dividing h. Curve25519 has h=8, so there are 8 points of order dividing 8:

- The identity O (order 1)
- Points of order 2, 4, 8

If an adversary sends one of these small-order points as their ECDH public key, the shared secret will be one of at most 8 possible values — leaking bits of the private key via oracle queries.

**Mitigation in X25519:** The "clamping" of private keys (clearing the bottom 3 bits and top 2 bits) makes the scalar always divisible by 8 and bounded below by 2^254. Multiplying a small-order point by a multiple of 8 always gives O — the shared secret is always O regardless of the small-order input, so the leak contains no information.

**secp256k1 (h=1):** All non-identity points have full group order n. No small-subgroup attack is possible. One of the practical advantages of prime-order curves.

### 8.4 The DUAL_EC_DRBG Backdoor

In 2007, NIST standardized Dual Elliptic Curve Deterministic Random Bit Generator (DUAL_EC_DRBG) as SP 800-90A. It uses two points P and Q on an elliptic curve:

```
s_{i+1} = (s_i · P).x   (x-coordinate of scalar mult)
output_i = (s_{i+1} · Q).x truncated to 240 bits
```

If the designer knows e such that Q = e·P (effectively, a "backdoor scalar"), then:

```
Given 30 bytes of output (240 bits), the designer can enumerate ~2^16 possibilities 
for the full x-coordinate of s_{i+1}·Q, recover s_{i+1}·Q, and then:

s_{i+1} = ((s_{i+1}·Q) · e^{-1}).x mod n
         = (s_{i+1} · (e·P) · e^{-1}).x
         = (s_{i+1} · P).x
```

This allows the secret holder to predict all future RNG outputs from 30 bytes of observed output. Essentially, the backdoor allows reading the entire RNG state.

Schneier called it out in 2007. In 2013, Snowden documents confirmed the NSA backdoor. RSA Security received $10M to make DUAL_EC_DRBG the default in their BSAFE product.

**Lesson:** Always demand "nothing up my sleeve" justification for curve/parameter choices. Bernstein's Curve25519 was specifically designed with transparent parameter generation.

### 8.4a AES-GCM Nonce Reuse: The Symmetric Counterpart

AES-GCM is TLS 1.3's primary cipher (TLS_AES_256_GCM_SHA384). GCM = Galois/Counter Mode — a stream cipher (CTR) combined with a polynomial MAC (GHASH over GF(2^128)).

**The GHASH MAC:**
```
auth_tag = GHASH(H, AAD, ciphertext) XOR E_K(nonce || 0...01)
where H = E_K(0^128)   (a fixed hash key derived from the cipher key)
```

**The catastrophe:** If the same (key, nonce) pair is used for two different messages m1 and m2:

```
ciphertext1 = m1 XOR CTR_K(nonce)
ciphertext2 = m2 XOR CTR_K(nonce)

ciphertext1 XOR ciphertext2 = m1 XOR m2   ← plaintext XOR recoverable!
```

Additionally, the attacker can forge valid authentication tags because GHASH is linear over GF(2^128). With two (ciphertext, tag) pairs under the same nonce, the polynomial structure can be solved to recover H, after which arbitrary messages can be authenticated without the key.

**TLS 1.3 defense:** The nonce for each AEAD operation is the record sequence number XOR'd with a per-key IV. Since sequence numbers are never reused within a session, nonces are never reused. TLS 1.3 also rotates keys periodically (KeyUpdate handshake message) to limit exposure.

**Lesson:** AEAD nonces have the same catastrophic consequences for nonce reuse as ECDSA — both produce keystream/keystream XOR attacks. AES-GCM has a 12-byte (96-bit) nonce — short enough that random nonces have birthday collision risk after ~2^48 messages with the same key. Production systems either use sequence-number nonces (TLS) or switch to XChaCha20-Poly1305 (192-bit nonces, safe for random generation).

### 8.5 Length-Extension Attacks on Merkle-Damgård

For Merkle-Damgård hashes (MD5, SHA-1, SHA-256):

**Given** H(m) and len(m) (without knowing m), an attacker can compute H(m || padding || suffix) for any suffix.

**Why?** H(m) is the internal state after processing m. By starting the hash computation from this state (instead of IV), and processing padding+suffix, the attacker gets a valid hash — without knowing m.

**Vulnerable pattern:**
```python
# BAD: MAC = H(secret || message) is vulnerable to length extension
mac = sha256(secret + message).digest()
# Attacker who knows mac and len(secret) can compute:
# sha256(secret + message + padding + attacker_suffix)
# without knowing secret!
```

**Fix — HMAC:**
```python
# GOOD: HMAC(K, m) = H(K XOR opad || H(K XOR ipad || m))
# The nested structure prevents length extension:
# Inner hash: H(ipad·K || m) — attacker can't extend this without K
# Outer hash: H(opad·K || inner) — completely different key padding
mac = hmac.new(secret, message, hashlib.sha256).digest()
```

**SHA-3 (Keccak sponge):** Immune to length extension because the output is not the full internal state — the "capacity" portion is hidden. Extending the hash would require knowing the hidden capacity bits.

---

## 9. Hard Exercises (With Solutions)

### 9.1 Exercise (a): ECDSA Nonce Reuse Key Recovery

**Setup:** Alice uses the same nonce k for two ECDSA signatures over messages m1, m2 (using secp256k1).

**Given:** H(m1), H(m2), s1, s2, r (same r because same k), n.

**Mathematical derivation:**

```
s1 = k^{-1}(e1 + r·d) mod n     where e1 = H(m1)
s2 = k^{-1}(e2 + r·d) mod n     where e2 = H(m2)

s1 - s2 = k^{-1}(e1 - e2) mod n
k = (e1 - e2)(s1 - s2)^{-1} mod n

d = (s1·k - e1)·r^{-1} mod n
```

```python
# ─────────────────────────────────────────────
# ECDSA Nonce Reuse Attack
# ─────────────────────────────────────────────

def ecdsa_nonce_reuse_attack(
    e1: int,   # H(m1): hash of first message
    e2: int,   # H(m2): hash of second message
    s1: int,   # first signature's s component
    s2: int,   # second signature's s component
    r: int,    # shared r (same nonce k was used)
    n: int     # group order
) -> tuple[int, int]:
    """
    Recover nonce k and private key d from two ECDSA signatures
    with the same nonce.
    
    Returns (k, d) — the nonce and the private key.
    
    This is what broke Sony PS3 and many other systems using
    broken ECDSA implementations.
    """
    if s1 == s2:
        raise ValueError("s1 == s2 would mean e1 == e2 — identical messages signed with same nonce")
    
    # k = (e1 - e2) / (s1 - s2) mod n
    k = (e1 - e2) * pow(s1 - s2, -1, n) % n
    
    # d = (s1·k - e1) / r mod n
    d = (s1 * k - e1) * pow(r, -1, n) % n
    
    return k, d


def demonstrate_nonce_reuse_attack():
    """
    Demonstrate the nonce reuse attack:
    1. Sign two messages with the same (deliberately reused) k
    2. Recover k and private key using just the public signatures
    """
    ecdsa = ECDSA(secp256k1, G, _n)
    
    # Alice's keypair
    alice_private = secrets.randbelow(_n - 1) + 1
    alice_public  = secp256k1.scalar_mul(alice_private, G)
    
    m1 = b"Transaction: pay Bob 1 BTC"
    m2 = b"Transaction: pay Charlie 2 BTC"
    
    # Compute message hashes
    e1 = ecdsa._hash_message(m1)
    e2 = ecdsa._hash_message(m2)
    
    # VULNERABILITY: reuse same k for both signatures
    k_reused = secrets.randbelow(_n - 1) + 1
    
    R = secp256k1.scalar_mul(k_reused, G)
    r = R.x % _n
    
    s1 = pow(k_reused, -1, _n) * (e1 + r * alice_private) % _n
    s2 = pow(k_reused, -1, _n) * (e2 + r * alice_private) % _n
    
    assert r != 0 and s1 != 0 and s2 != 0
    
    # Attacker observes (r, s1, s2) and the messages (public)
    # Runs the recovery attack:
    recovered_k, recovered_d = ecdsa_nonce_reuse_attack(e1, e2, s1, s2, r, _n)
    
    assert recovered_k == k_reused, "k recovery failed!"
    assert recovered_d == alice_private, "Private key recovery failed!"
    
    # Verify the recovered private key
    recovered_pub = secp256k1.scalar_mul(recovered_d, G)
    assert recovered_pub == alice_public, "Recovered key doesn't match public key!"
    
    print(f"✓ Nonce reuse attack succeeded!")
    print(f"  Original k  = {k_reused:#066x}")
    print(f"  Recovered k = {recovered_k:#066x}")
    print(f"  Original d  = {alice_private:#066x}")
    print(f"  Recovered d = {recovered_d:#066x}")
    print(f"  LESSON: NEVER reuse ECDSA nonces. Use RFC 6979 deterministic k.")


# demonstrate_nonce_reuse_attack()  # uncomment to run
```

### 9.2 Exercise (b): Full Schnorr Proof Test Suite

```python
def schnorr_complete_test_suite():
    """
    Complete Schnorr test suite as specified:
    1. Generate random secret
    2. Create proof
    3. Verify proof succeeds
    4. Tamper with s → verify fails
    5. Additional edge cases
    """
    schnorr = SchnorrProof(secp256k1, G, _n)
    
    print("=== Schnorr Proof System Test Suite ===\n")
    
    # Test 1: Generate and verify
    secret = secrets.randbelow(_n - 1) + 1
    Y, A, s = schnorr.prove(secret)
    assert schnorr.verify(Y, A, s), "Step 3: Proof should verify!"
    print(f"[1] ✓ Random secret generated: {secret.bit_length()} bits")
    print(f"[2] ✓ Proof created: A={A.x:#010x}..., s={s:#010x}...")
    print(f"[3] ✓ Proof VERIFIED successfully")
    
    # Test 4: Tamper with s by +1
    tampered_s = (s + 1) % _n
    result = schnorr.verify(Y, A, tampered_s)
    assert not result, "Tampered proof should fail!"
    print(f"[4] ✓ Tampered proof (s+1) REJECTED — ZK soundness holds")
    
    # Additional tampers
    for delta, label in [(-1,"s-1"), (2,"s+2"), (_n//2, "s+n/2")]:
        assert not schnorr.verify(Y, A, (s + delta) % _n), f"Tamper {label} should fail!"
    print(f"    ✓ Multiple s-tamperings all rejected")
    
    # Tamper commitment A
    _, A2, _ = schnorr.prove(secrets.randbelow(_n - 1) + 1)
    assert not schnorr.verify(Y, A2, s), "Wrong A should fail!"
    print(f"    ✓ Wrong commitment A rejected")
    
    # Tamper public key Y
    wrong_Y = secp256k1.scalar_mul(secret + 1, G)
    assert not schnorr.verify(wrong_Y, A, s), "Wrong Y should fail!"
    print(f"    ✓ Wrong public key Y rejected")
    
    # Edge case: secret = 1
    Y1, A1, s1 = schnorr.prove(1)
    assert schnorr.verify(Y1, A1, s1)
    print(f"    ✓ Edge case: secret = 1 works")
    
    # Edge case: secret = n-1 (max valid value)
    Yn, An, sn = schnorr.prove(_n - 1)
    assert schnorr.verify(Yn, An, sn)
    print(f"    ✓ Edge case: secret = n-1 works")
    
    # Multiple proofs for same secret are different (non-deterministic r)
    _, A_a, s_a = schnorr.prove(secret)
    _, A_b, s_b = schnorr.prove(secret)
    assert (A_a, s_a) != (A_b, s_b), "Two proofs should be different (randomized r)"
    assert schnorr.verify(Y, A_a, s_a) and schnorr.verify(Y, A_b, s_b)
    print(f"    ✓ Multiple proofs for same secret are distinct (randomized)")
    
    print(f"\n✓ All Schnorr tests passed!")


# schnorr_complete_test_suite()  # uncomment to run
```

### 9.3 Exercise (c): Forward Secrecy Analysis

**Question:** If an attacker steals the TLS 1.3 server's certificate private key, can they decrypt recorded past sessions?

**Answer: No.** Here is the precise argument using the key schedule:

**The key schedule dependency graph:**
```
early_secret = HKDF-Extract(0, PSK or 0)
     │
     ▼
handshake_secret = HKDF-Extract(
                     Derive(early_secret, "derived"),
                     shared_secret                    ← CRITICAL
                   )
     │
     ▼ (handshake_traffic_keys: encrypt Certificate, Finished)
     │
master_secret = HKDF-Extract(
                  Derive(handshake_secret, "derived"),
                  0
                )
     │
     ▼ (application_traffic_keys: encrypt all application data)
```

**The critical value `shared_secret`** is derived from ephemeral ECDH:

```python
shared_secret = X25519(server_ephemeral_private, client_x25519_public)
              = X25519(client_ephemeral_private, server_x25519_public)
```

**What the certificate private key is used for:**
```
CertificateVerify.signature = ECDSA_Sign(
    cert_private_key,
    SHA-256(transcript_hash)
)
```

The certificate private key is used **only** to sign the transcript. It is **not** used to derive `shared_secret`, `handshake_secret`, `master_secret`, or any traffic key.

**Therefore:** An attacker with the certificate private key can:
- ✓ Impersonate the server in new connections (forge CertificateVerify)
- ✓ Perform a MITM attack against clients that connect after the compromise

But cannot:
- ✗ Decrypt past sessions (would require ephemeral private keys, which are deleted after handshake)
- ✗ Compute `shared_secret` from ephemeral public keys alone (ECDLP hardness)

**Where exactly forward secrecy is established:** At the point where `server_ephemeral_private` and `client_ephemeral_private` are securely deleted (immediately after computing `shared_secret`). Both RFC 8446 and its implementing libraries (OpenSSL, BoringSSL) zero these values explicitly.

**The 0-RTT exception:** 0-RTT data uses a PSK-derived key, not ephemeral ECDH. If the PSK (from the previous session's resumption ticket) is compromised, 0-RTT data can be decrypted. Additionally, 0-RTT data can be replayed — there is no per-session proof of freshness. For high-security applications, 0-RTT should be disabled.

---

## Summary and Key Takeaways

| Concept | Implementation Pitfall | Correct Approach |
|---|---|---|
| Scalar multiplication | Data-dependent branches → timing attack | Montgomery ladder, blinding |
| ECDSA nonce k | Reuse or weak RNG → full key recovery | RFC 6979 deterministic k |
| ECDH public key | No validation → invalid curve attacks | Always check point is on curve, not O |
| Hash MACs | H(key \|\| msg) → length extension | HMAC (nested construction) |
| Curve parameters | Designer-chosen → backdoor possible | Nothing-up-my-sleeve generation |
| Forward secrecy | Long-term key in key derivation | Ephemeral DH, delete after use |
| Small subgroups | Cofactor h>1, no multiplying → leak | X25519 clamping, or verify order |

---

## 10. Further Exercises for Self-Study

These exercises require independent research beyond this lesson. No solutions provided — work through them to develop cryptographic intuition.

### 10.1 Threshold Signatures (t-of-n)
**(Hard)** Extend the Schnorr scheme to a **Shamir secret sharing** threshold signature where any t of n parties can sign, but fewer than t cannot:
1. Dealer splits private key x into n shares using degree-(t-1) polynomial over F_n.
2. During signing, t parties each contribute a partial signature using their share.
3. Final signature is Lagrange-interpolated from the partial responses.
4. Implement for t=2, n=3. Verify: any 2 of 3 parties can sign; no single party can.

### 10.2 Bilinear Pairings and BLS Signatures
**(Very Hard)** BLS signatures (Boneh-Lynn-Shacham) use a **bilinear pairing** e: G1 × G2 → GT where:
```
e(a·P, b·Q) = e(P, Q)^{ab}
```
This enables:
- **Signature aggregation:** n signatures → 1 signature (O(1) verification with n public keys)
- **Non-interactive**: no MuSig2-style nonce protocol needed
- Used in Ethereum 2.0 beacon chain (validator signatures)

Research question: Why does BLS use a different curve (BLS12-381) than secp256k1? What is the embedding degree and why does it matter for pairing security?

### 10.3 Attacking a Broken PRF
Suppose you have oracle access to F(k, x) where k is a 16-byte key and F is:
```python
def F(k: bytes, x: bytes) -> bytes:
    return hashlib.sha256(k + x + k).digest()[:16]
```
Show that this is NOT a secure pseudorandom function. Specifically, construct a distinguisher that identifies F from a random function using only 3 oracle queries, exploiting the length-extension property or structure of the construction.

### 10.4 Implementing Constant-Time Field Arithmetic
Implement a version of `modinv` and point addition that is genuinely constant-time in Python (using bitmasking instead of branching). Measure the timing variance on 10,000 random inputs and compare it to the variable-time implementation. Note: Python itself adds timing noise through garbage collection and interpreter overhead — identify which operations remain vulnerable even with bitmasked arithmetic.

### 10.5 Certificate Transparency and Merkle Inclusion Proofs
TLS certificates are logged in append-only Merkle trees (RFC 9162). Given a leaf and a Merkle proof (sibling hashes along the path to root), implement:
1. `verify_inclusion(leaf, proof, root) → bool`
2. Explain why a dishonest log cannot equivocate (present two different views of the tree) without being detected, given that multiple auditors independently verify the root hash.

### Further Reading

- Boneh & Shoup, *A Graduate Course in Applied Cryptography* (free, crypto.stanford.edu)
- Bernstein & Lange, *SafeCurves: choosing safe curves for elliptic-curve cryptography*
- RFC 8446 (TLS 1.3), RFC 6979 (Deterministic ECDSA), RFC 7748 (Curve25519/X25519)
- Kocher (1996), *Timing Attacks on Implementations of Diffie-Hellman, RSA, DSS, and Other Systems*
- Bernstein (2006), *Curve25519: New Diffie-Hellman Speed Records*
- NIST FIPS 186-5 (Digital Signature Standard), FIPS 180-4 (SHA standard)

---

*Lesson 07 of the Applied Cryptography Series. All code is for educational purposes. Do not use these implementations in production — use audited libraries (libsodium, BoringSSL, or language-native crypto with hardware acceleration).*
