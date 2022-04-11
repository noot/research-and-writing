# ECDSA Adaptor Signatures

Overview:
- adaptor signatures (or signature adaptors) are a modified version of the usual digital signatures we know of
- also referred to as "one-time verifiably encrypted signatures" or one-time VES
- it adds an additional parameter `Y = y*G` where `y` is a private scalar (referred to as the encryption secret key)
- when an adaptor signature is combined with a normal signature, the private scalar can be calculated
- additionally, combining the adaptor with the encryption secret `y` reveals a standard signature
- thus the adaptor can be thought of as an encrypted signature (ie. ciphertext) where `Y` is the encryption public key, and the standard signature is the plaintext

Usages:
- revealing an encryption key to some message only once a signature is published
- coinswap protocols (ie. when a transaction is submitted to a blockchain, some other party with an adaptor can uncover the secret value)
- bets placed on the result of some real-world event signed by an oracle, without the oracle needing to know of the bet

## ECDSA signatures: high level API

### Signing: `ecdsa_sign`

Input:
- `x`: signing private key
- `message`: message to sign

Output:
- `(r, s)`: scalar tuple

### Verification: `ecdsa_verify`
Input:
- signing output, public key`P = x*G`, `message`

Output: 
- true or false

## ECDSA adaptor signatures: high level API

### Signing:`ecdsa_adaptor_sign`
Input:
- `x`: signing private key
- `Y`: encryption public key
- `message`: message to sign

Output:
- DLEQ proof `(b, c)`
- adaptor signature `(R, R_a, s_a)`

### Verification: `ecdsa_adaptor_verify`
Input:
- adaptor signature and DLEQ proof
- `P`: public signing key
- `Y`: encryption public key
- `message`: signed message

Output: 
- true or false

### Secret encryption key recovery: `ecdsa_adaptor_recover`
Input:
- `Y`: encryption public key
- adaptor signature
- standard ECDSA signature 

Output:
- `y`: secret decryption key OR error

### Signature decryption: `ecdsa_adaptor_decrypt`
Input:
- adaptor signature
- `y`: decryption secret key

Output:
- `(r, s)` ECDSA signature

# The math

I will be discussing ECDSA signature adaptors. Schnorr signature adaptors also exist. Firstly, we will discuss ECDSA signatures, then the adaptors, as we need to understand both to understand how the secret is revealed.

Assume we have some elliptic curve, where `n` is the curve order and `G` is the generator point. Scalars are integers between `0...n-1`. Points are points on the curve with coordinates `(x, y)` which are members of the curve group. 

Scalars have four operators between themselves: addition, subtraction, multiplication, and inversion, all done `mod n`.

Points have two operators between themselves: addition and subtraction. 

Points and scalars have one operation between each other: multiplication, ie. multiplying a point by a scalar.

We also have an operation `hash_to_scalar(m)` which hashes some data `m` and returns a scalar between `0...n-1`.

## ECDSA signatures

### Signing: `ecdsa_sign`

Let's define a few variables:
> `m` := message
> `z` := `hash_to_scalar(m)` 
> `k` := random scalar
> `x` := private key (*a scalar*)
> `P` := `x * G` (*public key, a point on the curve*)

To create a signature of some message `m`:
> `R = k*G`
> `r = x-coordinate of R`
> `s = (z + r*x) / k`

The signature is then `(r, s)`. This results a signature with size = 32 + 32 = 64 bytes. Optionally +1 byte for key recovery known as `ecrecover` for Ethereum. This byte is known as the `V` parameter and is used to signify the y-coordinate that corresponds to `r`, which is only an x-coordinate. This is needed since each x-coordinate on the curve has 2 corresponding y-coordinates, so `V` tells you whether it is the upper or lower one.

### Verification: `ecdsa_verify`
To verify the signature given `(r, s)`, `P`, and `m`:
> `s x R' = r x P + z x G`
> `R' = (1/s)(r x P + z x G)`
> if x-coordinate of `R'` is  `r`, then the signature is valid. otherwise, it is invalid.

https://www.programmersought.com/images/767/f85a198a24cfad8518a50993ad112357.JPEG

## ECDSA adaptor signatures

### DLEQ Proof

Since adaptor signatures contain `R = k * G` and `R_a = k * Y` where `k` is a random scalar and `Y` is the encryption key, there needs to be a proof of equality that the `k` value is the same for both `R` and `R_a`.

This is referred to as a proof of **discrete logarithm equality** or a DLEq proof. It proves that given `X = a * G` and `Z = b * Y`, `a == b`.

#### Proving: `dleq_prove`

Define:
> `w` := non-zero scalar representing proof witness
> `(X, Y, Z)` := three non-zero points defining the statement to be verified

To create the DLEq proof:
> `k` := random scalar
> `k_G` := `k * G`
> `k_Y` := `k * Y`
> `b` := `hash_to_scalar(X || Y || Z || k_G || k_Y)`
> `c` := `k + w*b`

The proof is the tuple `(b, c)`.

#### Verification: `dleq_verify`

Given the proof `(b, c)` and the points `(X, Y, Z)`:
> `k_G` := `c*G - b*X`
> `k_Y` := `c*Y - b*Z`
> `b'` := `hash_to_scalar(X || Y || Z || k_G || k_Y)`
> if `b' == b`, then the proof is valid. otherwise, it is invalid.

### Adaptor signing: `ecdsa_adaptor_sign`

Define:
> `x`:= signing secret key
> `Y`:= encryption public key
> `m` := message

To create the signature:
> `k` := random scalar
> `z` := `hash_to_scalar(m)`
> `k_G` := `k * G`
> `k_Y` := `k * Y`
> `r` := x-coordinate of `k_Y` mod n
> `s` := `(1/k)(z + r*x)`
> `dleq_proof` := `dleq_prove(k, (k_G, Y, k_Y))`

The signature is the values `(k_G, k_Y, s, dleq_proof).`

### Adaptor verification: `ecdsa_adaptor_verify`

Given adaptor signature `(k_G, k_Y, s, dleq_proof)`, public key `P`, public encryption key `Y`, and the message `m`:
> check `dleq_verify(dleq_proof, (k_G, Y, k_Y))` 
> `z` := `hash_to_scalar(m)`
> `r` := x-coordinate of `k_Y` mod n
> `u1` := `z / s`
> `u2` := `r / s`
> if `u1*G + u2*P == k_G`, then the signature is valid, otherwise, it is invalid.

### Secret encryption key recovery: `ecdsa_adaptor_recover`

Given adaptor signature `(k_G, k_Y, s_a, dleq_proof)`, public encryption key `Y`, and the ECDSA signature `(r, s)`:
> `y` := `s_a / s`
> `Y'` := `y * G`
> if `Y' == Y` return `y`
> else if `Y' == -Y` return `-y`
> otherwise fail

### ECDSA signature decryption: `ecdsa_adaptor_decrypt`

Given the adaptor signature `(k_G, k_Y, s_a, dleq_proof)` and the encryption secret `y`, a valid ECDSA signature can be decrypted.
> `s` := `s_a / y`
> `r` := x-coordinate of `k_Y` mod n

The ECDSA signature is then `(r, s)`.


# Additional resources
- ECDSA adaptor signature specification: https://github.com/discreetlogcontracts/dlcspecs/blob/master/ECDSA-adaptor.md
- one-time VES (Schnorr and ECDSA): https://github.com/LLFourn/one-time-VES/blob/master/main.pdf
- Schnorr adaptor signatures and bitcoin atomic swaps: https://github.com/ElementsProject/scriptless-scripts/blob/master/md/atomic-swap.md