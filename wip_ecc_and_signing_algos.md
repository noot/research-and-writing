# An intro to elliptic curve cryptography and signing algorithms used in blockchain (ECDSA, Schnorr, BLS)
#### Elizabeth
#### June 2021

## Elliptic curve cryptography

In this article, I will introduce some of the preliminary math concepts needed to understand elliptic curve cryptography and some signature algorithms at a high level.

To start off with preliminaries: 

> a **finite field** is a finite set of numbers, for example [0...n-1] where n is some prime, which may be defined as integers mod n.
> Additionally, the addition (+), subtraction (-), multiplication (x), and division (/) operators are defined over this field.
> the size of the set is called the **order**.
> a **group** is a set that has an operation defined over it which combines two elements associatively to form a third element also in the set, and additionally has an **identity element** and **inverse elements**.

To define a field, we can use modular arithmetic. For example, say we want a field with order 7. Then we can define our field as the positive integers *modulo 7*. (Positive integers are `[0, 1, 2, 3, ... so on]`.) Then, our set is `[0, 1, 2, 3, 4, 5, 6]`. 

Some examples of addition over this field:
> 0 + 6 mod 7 = 6
> 5 + 6 mod 7 = 11 mod 7 = 4

Some examples of subtraction over this field:
> 6 - 1 mod 7 = 5
> 5 - 6 mod 7 = -1 mod 7 = 6

Some examples of multiplication over this field:
> 6 x 1 mod 7 = 6
> 5 x 6 mod 7 = 30 mod 7 = 2

Some examples of division over this field:
> 6 / 1 mod 7 = 6
> 6 / 2 mod 7 = 3

As you can see, all the results fit nicely into our set `[0, 1, 2, 3, 4, 5, 6]` by using the `modulo 7` operation. Nice!

Let's also show this is a group. We already have our addition operation, so let's show that it's associative, ie. `a + (b + c) = (a + b) + c`:

> 1 + (3 + 5) mod 7 = 2
> (1 + 3) + 5 mod 7 = 2

The **identity element** is an element such that when added to another element of the set, the result is the same element, ie. the number 0.
An **inverse element** exists for each element `a` is such that `a + b = 0`, where `b` is the inverse element. For example:
> 6 + 1 mod 7 = 0

Thus over our field, the inverse of 1 is 6.

We can extend the example just shown to define fields in general that have an order that is some large prime number.  We will see later that this allows the field to have some nice crypto-friendly properties.
 
Elliptic curves are defined by the equation `y^2 = x^3 + ax + b`. It turns out that when certain `a`, `b` parameters are chosen, as well as a specific field to define the curve over, we are given a set of points which have certain properties that allow them to work well for cryptographic purposes.

Aside: You may have heard about curves such as ed25519 and secp256k1. These are elliptic curve with specific parameters that are defined over a finite field with a specific order, which is some very large prime number.

A couple more definitions:
> **scalar**: a real number, eg. an integer
> **point**: a point on a curve, consists of two scalars; has the form (x-coordinate, y-coordinate)

Each curve has a special number called the **generator** (or base point) which we will denote **G**. This parameter also defined by the curve. The generator is chosen a certain way. A base point can be any potentially be element in the set; however, for cryptgraphic purposes, we want a point that when added to itself many times, *does not cycle for a large n*. In other words, we want to be able to perform `GxG` `n` times, where `n` is the greatest value possible such that `G^n != G`. ie. `G`, `2xG`.. `nxG` are all not equal for a very large n.

Why is the generator chosen this way? This is because private keys are defined as some number x within a set `[0, ..., n-1]` and the public key is `x*G`. We want the curve to have as many private/public combinations as as possible, as duplicate keys are extremely undesirable.

Why is cryptography "secure"? In other words, given a public key `P = x*G mod n`, why are we not able to solve for the private key `x`? This relies on a problem referred to as the **discrete logarithm problem**. In the case `P = x*G mod n`, the discrete logarithm problem can be expressed as solving for `x = P/G mod n`.  For the groups that are used for ECC, solving this is hard (ie. infeasible), as there is no efficient algorithm to solve for this. Note: this is often written as `P = G^x mod n` and `x = log_G(P) mod n`, hence why is it the discrete *logarithm*. [0](https://www.doc.ic.ac.uk/~mrh/330tutor/ch06s02.html)

## Why elliptic curves?

Why is elliptic curve cryptopgraphy widely used by blockchains (and other domains) as opposed to other signing schemes like RSA?

> As with elliptic-curve cryptography in general, the bit size of the public key believed to be needed for ECDSA is about twice the size of the security level, in bits [1]. For example, at a security level of 80 bits—meaning an attacker requires a maximum of about {\displaystyle 2^{80}}2^{80} operations to find the private key—the size of an ECDSA private key would be 160 bits, whereas the size of a DSA private key is at least 1024 bits. On the other hand, the signature size is the same for both DSA and ECDSA: approximately {\displaystyle 4t}4t bits, where {\displaystyle t}t is the security level measured in bits, that is, about 320 bits for a security level of 80 bits. [1](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm#Key_and_signature-size)

## ECDSA
Let's move on to our signing algorithms. The most well-known is likely ECDSA or the **Elliptic Curve Digital Signature Algorithm**. This is the algorithm used by Bitcoin, Ethereum, and various other blockchains. 

Let's firstly go over the algorithm. Let's define a few variables:
> `m` := message
> `z` := `hash(m)` (*note: this is a hash function that results in a scalar value*)
> `k` := random scalar
> `pk` := private key (*a scalar*)
> `P` := `pk x G` (*public key, a point on the curve*)

To create a signature of some message `m`:
> `R = k*G`
> `r = x-coordinate of R`
> `s = (z+r*pk) / k`

The signature is then `(r, s)`. This results a signature with size = 32 + 32 = 64 bytes. Optionally +1 byte for key recovery known as `ecrecover` for Ethereum. This byte is known as the `V` parameter and is used to signify the y-coordinate that corresponds to `r`, which is only an x-coordinate. This is needed since each x-coordinate on the curve has 2 corresponding y-coordinates, so `V` tells you whether it is the upper or lower one.

To verify the signature:
> `s x R = r x P + z x G`
> `R = (1/s)(r x P + z x G)`
> if x-coordinate of `R == r`, then the signature is verified. otherwise, it is invalid.

https://www.programmersought.com/images/767/f85a198a24cfad8518a50993ad112357.JPEG

ECDSA is widely used, and for good reason. 
The pros of ECDSA are:
- It's simple and works on secure curves like secp256k1.
- The random scalar `k` is actually deterministic (RFC6979) and is based on message and private key, so the algorithm doesn't require a RNG.
- Can do key recovery, which is very nice for blockchain contexts. An ECDSA signature implicity transmits the public key (and thus blockchain address) without needing to transmit the 33-byte or 64-byte public key, which saves space.

However, ECDSA does have some cons:
- It requires inversion of s, which is expensive. (TODO: expand)
- No multisig support. (TODO: maybe there is but it's too complicated?)
    - Found this for threshold signatures: https://eprint.iacr.org/2019/114.pdf

## Schnorr signatures
The Schnorr signature algorithm is also a well-known signature algorithm, currently used by the Polkadot network, implemented in the schnorrkel library. (TODO: link) It has also been proposed that Bitcoin use schnorr signatures instead of ECDSA due to its smaller and more flexible multisig capabilities.

Let's go over the algorithm:
> `m` := message
> `pk` := private key
> `k` := random scalar
> `P` := `pk × G`
> `hash(x)` := some hash function that results in a scalar

To create a signature:
> `R = k × G`
> `s = k + hash(P, R, m) x pk`

The resulting signature is `(R, s)` where `R` is a point on the curve and can be expressed as in compact form using 33-bytes. Thus the size of the signature is = 33 + 32 bytes = 65 bytes.

To verify a signature:
> check that: `s × G == R + hash(P, R, m) × P`

https://www.programmersought.com/images/196/ebf26f2ea8713992d4590a49006ac524.JPEG

The algorithm itself is nice and straightforward and easily lends itself to batch verification, where many signatures can be combined and verified in one step. For example, the batch verification of (0...n) signatures is:
> (s_0 +...+ s_n) x G = (R_0 +...+ R_n) + (hash(P_0, R_0, m_0) x P_0 +...+ hash(P_n, R_n, m_n) x P_n TODO: add x_i random scalar to each term?

All the `s_i`, `R_i`, and `hash(P_i, R_i, m_i) x P_i` values are simply added up in the verification equation.

The pros of schnorr are:
- multisig support (TODO: add multisig algorithm) // Description or just list?
    - Multisig: Musig2 (https://github.com/ZenGo-X/multi-party-schnorr/blob/master/papers/musig2_simple_two_round_schnorr_multi_signatures.pdf) and Micali, et al. (https://github.com/ZenGo-X/multi-party-schnorr/blob/master/papers/accountable_subgroups_multisignatures.pdf)
    - Threshold signature: Stinson-Strob (https://github.com/ZenGo-X/multi-party-schnorr/blob/master/papers/provably_secure_distributed_schnorr_signatures_and_a_threshold_scheme.pdf)
- no inversion, faster to verify than ECDSA
- batch verification is simple and non-costly as it only consists of additions (basically free) and `n+1` point multiplications

The cons of schnorr are:
- multisig requires several communication rounds
- can't combine all signatures in a block into one (TODO: what did I mean by this)
- requires a RNG (random number generator) (TODO: why??)
    - Lol not entirely sure. Maybe because the randomness has to be unique for each signature to keep the private key secure, so maybe you need a RNG for that? Not really sure but was looking at https://crypto.stackexchange.com/a/13141

## BLS signatures

BLS is a comparatively newer signature scheme that leverages properties of certain curves, namely curves that support **pairings**.  A pairing is a property that certain curves have that allows for some nice mathematical results:

> given curve points `P`, `Q`, there exists a map function `e`: `e(P, Q) -> n`
> special property: e(xP, Q) = e(P, xQ) where x is a secret number, pairing does not reveal secret
> pairings are associative, ie. e(aP,bQ) = e(abP, Q) etc

However, only certain curves are pairing friendly, secp256k1 is not pairing-friendly. (TODO: name some pairing friendly curves such as BLS12-381)

We also need to define an additional function for BLS signatures:
> hash-to-curve function `H(m)`: In previous algorithms, the message is hashed and the hash is simply used as a number. For BLS we need a special "hash-to-curve" function which hashes a message directly to a point on the curve. One potential implementation of this is to hash the message, with some nonce, and use the result as a x-coordinate. If it maps to a point, then we are good, otherwise increment the nonce and try again.

To create a signature:
> `S = pk x H(m)`
The resulting signature `S` is a point on the curve, and is only 33 bytes!

To verify a signature, we utilize the properties of pairings:
> check that: `e(P, H(m)) == e(G, S)`

This works because: `e(P, H(m)) = e(pk×G, H(m)) = e(G, pk×H(m)) = e(G, S)`

To perform batch signature verification:
> `S = S+0+…+S_n`
> `e(G, S) = e(P_0, H(m_0))⋅e(P_1, H(m_1))⋅…⋅e(P_n, H(m_n))`

This is because (using the properties of pairings outlined above):
> `e(G, S) = e(G, S+0+…+S_n) = e(G, S)⋅e(G, S_1)⋅…⋅e(G, S_n) = e(G, pk_0×H(m_0))⋅…⋅e(G, pk_n×H(m_n)) = e(pk_0×G, H(m_0))⋅…⋅e(pk_n×G, H(m_n)) = e(P_0, H(m_0))⋅…⋅e(P_n, H(m_n))`

The pros of BLS:
- pairings (TODO: are they efficient?)
- signatures only take up 33 bytes, smaller than ECDSA and schnorr
- most diverse for signature aggregation options, makes it ideal for voting in consensus algorithms, threshold signatures, etc

The cons of BLS signatures:
- only works on specific curves (which are newer and may be less secure TODO: reference?)
- signing and verification is slower


## Current blockchain usages

### ECDSA (eth1, bitcoin)

TODO
- bitcoin and ethereum use the secp256k1 curve
- why did bitcoin choose secp256k1? basically unknown why Satoshi chose it, but potentially because it's optimized for scalar multiplications.
- why did ethereum choose secp256k1? because of interop with bitcoin 

### BLS (eth2)

TODO
 
### Schnorr (Polkadot)
- Polkadot uses **schnorrkel** which implements schnorr signatures (amongst other things like VRFs) over the Ristretto compression of Curve25519
- schnorrkel uses Curve25519 because it is (maybe) more secure than secp256k1 and allows for easier implementation of schnorr signatures
- schnorr was chosen over BLS as it allows for batch verification and multisigs, but can be implemented over more secure curves
- schnorr was chosen over ECDSA because of simpler batch verification
- see also: https://forum.w3f.community/t/account-signatures-and-keys-in-polkadot/70

## Additional resources
https://medium.com/cryptoadvance/how-schnorr-signatures-may-improve-bitcoin-91655bcb4744
https://medium.com/cryptoadvance/bls-signatures-better-than-schnorr-5a7fe30ea716
https://medium.com/asecuritysite-when-bob-met-alice/picking-a-base-point-in-ecc-8d7b852b88a6
https://wiki.polkadot.network/docs/en/learn-keys#faq
https://w3f-research.readthedocs.io/en/latest/polkadot/keys/1-accounts-more.html#:~:text=We%20prefer%20Schnorr%20signatures%20because,the%20Ed25519%20curve%20or%20secp256k1.
https://asecuritysite.com/encryption/ecc_points_real?a0=2&a1=3&a2=97
https://bitcoincore.org/en/2017/03/23/schnorr-signature-aggregation/
https://bitcoin.stackexchange.com/questions/34288/what-are-the-implications-of-schnorr-signatures
https://forum.w3f.community/t/account-signatures-and-keys-in-polkadot/70