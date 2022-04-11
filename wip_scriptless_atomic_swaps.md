# Scriptless Cross-Chain Atomic Swaps

# 1. Introduction

A scriptless cross-chain atomic swap is one that requires no scripting capabilities on either chain. This solves many limitation of existing cross-chain atomic swaps, including but not limited to:
- scripting capabilities built in to the chain
- transaction indistinguishability: swap transactions should be indistinguishable from other transactions
- requiring that a party on a specific chain must move first; for example, with BTC-XMR or ETH-XMR atomic swaps, the BTC or ETH holder must lock their funds first. 

# 2. Preliminaries

## 2.1 Adaptor signatures (One-time VES)

See https://hackmd.io/Gdf69v-gTBm0Dhmu6tAF_A?view#ECDSA-adaptor-signatures-high-level-API for now. Functions used: `ecdsa_adaptor_sign`, `ecdsa_adaptor_verify`, and `ecdsa_adaptor_recover`.

## 2.2 Verifiable timed signatures 

A verifiable timed signature or VTS is an encrypted signature, where the decrypted signature can revealed after some time `t`. It can be implemented for ECDSA, Schnorr, or BLS signature schemes.

API:
- `vts_sign(msg, t, pk)`: create an encrypted timed signature with the secret key `pk` and message `msg` that unlocks after time `t`.
- `vts_verify(msg, P)`: verify an encrypted timed signature with the public key `P` and message `msg`.
- `vts_unlock(enc_sig)`: unlock an encrypted signature `enc_sig` after time `t`.

# 3. Protocol

## 3.1 Situation

Alice and Bob wish two swap two coins, A and B, in the amount `v_a` and `v_b`. Alice holds `v_a` and wants `v_b`, and Bob holds `v_b` and wants `v_a`. We assume Alice and Bob have met and agree to perform the swap.

## 3.2 Happy Path

Happy path:

1. Alice generates a secret `x_a` and the corresponding public key `P_a`. She sends Bob a message containing `P_a`.
2. Bob receives the message and creates an adaptor signature `adaptor_transfer` := (`r_a, s_a, proof, Y`) where `Y = x_b * G` is an encryption key with the secret `x_b`. The adaptor signature message is a transaction `v_a` moving out of the account `AccLockA` with public key `Y`. When combined with a normal ECDSA signature `sig_transfer` := `(r, s)`, the secret "decryption key" `x_b` can be revealed. He also locks `v_b` in the account with the public key `AccLockB = x_b * P_a`.
3. Alice verifies the adaptor signature `adaptor_transfer` and that the amount `v_b` locked is correct. If so, Alice locks `v_a` in the account `AccLockA = Y`, which is owned by Bob.
4. If Bob moves any ether out of the account `AccLockA = Y`, he publishes an ECDSA signature `(r, s)` which Alice can then combine with `adaptor_transfer` sent in step 2 to reveal the secret value `x_b`. 
5. Then, Alice owns the private key `x_b * x_a` to the account `AccLockB` and can sign a transaction that originates from that account.

## 3.2 Refund Path

Unhappy path:
- In addition to the public keys exchanged by Alice and Bob, Bob also sends Alice a VTS that unlocks after time `t` that transfers the funds locked by Bob in `AccLockB` back to Bob. This decrypted signature is referred to as `sig_refund`.
- In the case Alice locks `v_a` and Bob goes offline until time `t`, Alice can return `v_a` to herself by obtaining `sig_refund`, publishing it to chain  B, refunding `v_b` to Bob, while also combining `adaptor_b` with `sig_refund` revealing `x_b`, allowing her to obtain the account containing `v_a`.

## 3.3 Problem

This protocol is not secure. In step 4 when Bob publishes the transaction to move funds out of `AccLockA`, Alice can find the secret `x_b` while the transaction is still in the mempool and front-run Bob, thus obtaining both `v_a` and `v_b`. Similarly in the refund case, there is nothing stopping Alice from claiming both `v_a` and `v_b`.

A potential solution to this makes the protocol non-scriptless: create a contract on-chain that enforces that only an address owned by Bob can withdraw funds from `AccLockA`. This address does not need to be the one with public key `Y`. If the address is not the one with public key `Y`, Bob must also submit a valid ECDSA signature for the message signed by `adaptor_transfer`, which is verified by the contract. This requiries scripting capabilities on chain `A`.