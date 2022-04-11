# Cross-chain Atomic Swaps between Ethereum and Monero
March 2022

## 0. Abstract

Atomic swaps are a method for trustlessly swapping assets between two blockchains. They have been implemented between blockchains that have capability for hash timelocked contracts; however, when one of the chains does not have any scripting capabilities, such as Monero, it introduces new challenges. A BTC-XMR atomic swap protocol has already been described and implemented. This article describes an ETH-XMR atomic swap protocol that utilizes the smart contract capabilities of Ethereum. Taking into account Ethereum's gas cost, it also uses off-chain discrete logarithm equality proofs as a method for reduction of transaction fees. 

## 1. Background

Atomic swaps are a method for trustlessly swapping assets between two blockchains. It is entirely peer-to-peer and requires no third party. Assuming the two participants are able to meet and agree on the amounts to swap, the protocol is able to proceed to completion. If the protocol is not able to complete, for example if one party goes offline part-way through, both parties will be refunded.

This article describes a protocol for performing an atomic swap between Ethereum and Monero. However, it can be generalized to work between any chain with smart contract capabilities equivalent to Ethereum, and any other chain without any such capabilities.

## 2. Situation

Say we have two parties, Alice and Bob, where Alice owns ETH and wants XMR, and Bob owns XMR and wants ETH. Alice and Bob meet and agree on the amounts to swap, which we will refer to as `eth_amount` and `xmr_amount`. The method for them to meet is out of the scope of this article; however, a distributed hash table is one possible way for them to discover each other in a decentralized manner.

## 3. Preliminaries

### 3.1 Ethereum

As part of its state transition, Ethereum executes its transactions through the Ethereum Virtual Machine (EVM), a nearly-Turing complete machine (the reason it is not is due to "gas", which puts an upper bound on the amount of operations a transaction can do). Thus, any execution that will not exceed the block gas limit can be performed. Gas is a measure of how expensive a transaction will be in terms of computing cost. Every instruction in the EVM has an associated gas cost based off of how expensive the computation. The financial cost of a transaction is the gas required multiplied by the "gas price", which is the cost per unit of gas in ETH.

Ethereum uses the secp256k1 curve and ECDSA as its signing algorithm. Additionally, ECDSA has the property of key recovery; one can recover the signing public key given a signature. (TODO; mention ecrecover hack?)

Unfortunately, Ethereum has no layer-1 privacy. Every transaction is publicly visible, including sender and recipient addresses, value transfers, smart contract calls, and smart contract bytecode. This limitation will be further discussed in section 5. 

### 3.2 Monero

Monero is a privacy-preserving blockchain that obfuscates the link between sender and recipient, as well as the value transferred. It has no smart contract or scripting capabilities. 

Unlike Ethereum, Monero uses the ed25519 curve and signing algorithm (TODO: double check signing algo?). 

### 3.3 Discrete logarithm equality proof

As part of the protocol, we wish to verify that a public key on the ed25519 curve has the same discrete logarithm as a public key on the secp256k1 curve. A discrete logarithm equality proof is used to prove equivalence of the secret key corresponding to public keys on either curve.

The details of the implementation are out of the scope of this paper, but are outlined in [3].

The high-level API used in the protocol is as follows:
- `dleq_prove(x) -> (P_a, P_b, proof)` where `x` is a scalar and the discrete logarithm of `P_a` and `P_b` on the secp256k1 and ed25519 curves respectively.
- `dleq_verify(P_a, P_b, proof) -> (true, false)`

## 4. Protocol

In the follow sections, I will describe an ETH-XMR atomic swap protocol between two parties, Alice and Bob, where Alice has ETH and wishes to swap for XMR, and Bob has XMR and wishes to swap for ETH. 

### 4.1 On-chain functionality

In the first step of the protocol, Alice and Bob exchange newly-generated public keys on the ed25519 and secp256k1 curves. For this section, we will refer to only the secp256k1 public keys, as they will be stored on-chain. We will refer to Alice's secp256k1 public key as `P_a`, with a corresponding private key `x_a`. Similarly, Bob has a public key `P_b` and a private key `x_b`.

Additionally, the swap requires two timeouts, `t0` and `t1` where `t0` < `t1`. 

`alice_address` and `bob_address` are the participants's original Ethereum addresses, which should not correspond to the swap keys `P_a` and `P_b`.

This protocol requires a smart contract to be deployed to the Ethereum blockchain that contains the following functionality:

- `initiate(alice_address, bob_address, P_a, P_b, t0, t1, value)`: Alice is able to initiate the swap on-chain by storing `P_a` and `P_b`. She also specifies two timeouts `t0` and `t1`. In the same step, she also pays the contract the agreed upon ETH swap value. 
- `set_ready()`: After this is called, the contract allows Bob to call the `claim(x_b)` function, while simultaneously disallowing Alice from calling `refund(x_a)`. Only Alice is able to call this function from `alice_address`. This function does not need to be called, as if `t0` passes, the contract acts the same as if `set_ready()` was called.
- `claim(x_b)`: This function is only callable by Bob from `bob_address`. Bob must provide the private key to `P_b` stored in the contract. The contract performs a scalar basepoint multiplication to verify that `x_b` corresponds to `P_b`. If the verification succeeds, `value` ETH is transferred to Bob. Otherwise, the transaction reverts. This function is only callable after `t0` or `set_ready()` is called, and before `t1` passes.
- `refund(x_a)`: This function is only callable by Alice from `alice_address`. ALice must provide the private key to `P_a` stored in the contract. The contract performs a scalar basepoint multiplication to verify that `x_a` corresponds to `P_a`. If the verification succeeds, `value` ETH is transferred to Alice. Otherwise, the transaction reverts.

### 4.2 Protocol steps

#### 4.2.1 Success path



#### 4.2.2 Refund path

### 4.3 On-chain ed25519 public key verification

## 5. Limitations

### 5.1 Privacy concerns

The current protocol has two obvious privacy concerns. On the Monero side, as all transactions are private by default, the receiver of the Monero has their privacy protected. However, on the Ethereum side, all transaction details are visible. Since the swap contract is deployed on-chain, even if the source code is not uploaded on a block explorer like Etherscan, it is still evident which accounts are interacting with the swap. Timing attacks can then be used to link an Ethereum account to a Monero account. Additionally, centralized parties can choose to ban accounts that interact with the atomic swap contract.

The second main concern is that the XMR-holding party must own ETH to call the swap contract's `Claim` function, as they need to pay for the gas for the transaction. This links the ETH claimer's existing account to the account that redeems the ETH from the swap contract. The user may use an existing ETH mixer such as Tornado cash to withdraw the funds from a fresh account, but there is still the overhead of requiring them to have access to ETH in the first place.

Section 6.1 aims to provide a high-level overview of the next iteration of the protocol which solves both these issues.

### 5.2 Griefing attacks

The protocol currently requires the ETH holder to move first, as if the XMR holder moves first, the ETH holder can simply go offline and the XMR is locked forever. This requires implementations to make the XMR holder the swap "maker"; ie. an XMR holder to wishes to swap for ETH must advertise a swap offer. Then, an ETH holder can act as the "taker", taking one of the existing offers by locking their ETH. Assuming the XMR holder actually holds the XMR they have advertised and is willing to follow the protocol, this scenario works. However, if the offer maker does not actually hold the XMR they claim to, or they go offline right after the offer is taken, then the ETH holder has no option but to refund their ETH. In this case, the ETH holder has lost some ETH due to the gas fees of initiating the swap as well as refunding, while the XMR holder has lost nothing.

### 5.3 Ethereum gas costs

In recent times, the gas price of Ethereum has reached record highs. (TODO: add estimates of what the swap would cost at certain gas prices)

## 6. Protocol V2

In this section, I will describe an ETH-XMR atomic swap protocol that uses adaptor signatures to greatly reduce gas fees.

### 6.1 Preliminaries

#### 6.1.1 Adaptor signatures


### 6.2 Protocol description

#### 6.2.1 On-chain functionality

```solidity
contract SwapV2 {
    address payable owner;
    address payable claimer;
    
    uint256 timeout_0;
    uint256 timeout_1;
    
    constructor(address _claimer, uint256 _timeoutDuration) public payable {
        owner = msg.sender;
        claimer = _claimer;
        timeout_0 = block.timestamp + _timeoutDuration;
        timeout_1 = block.timestamp + (_timeoutDuration * 2);
    }
    
    function claim() public {
        require(msg.sender == claimer);
        require(block.timestamp < timeout_1 && block.timestamp > timeout_0);
        claimer.transfer(this.value);
        selfdestruct();
    }
    
    function refund() public {
        require(msg.sender == owner);
        require(block.timestamp > timeout_1 || block.timestamp < timeout_0);
        owner.transfer(this.value);
        selfdestruct();
    }
}
```

#### 6.2.2 Success path

1. Alice and Bob meet and generate ed25519 keypairs, where `x_a` and `x_b`are their respective secret keys, and `P_a` and `P_b` are their respective public keys.
2. Alice prepares to lock her ETH in a smart contract by determining the `CREATE2` address where the contract will be deployed to. If the contract is a "factory" contract where the address is already known, this step can be omitted.
3. Alice sends Bob the swap contract address. Bob prepares a transaction `tx_claim` that calls `claim`. (TODO: how do we handle nonce?) He creates an adaptor signature `claim_adaptor` on `tx_claim` with the public encryption key `Y_b`. He sends this adaptor, his Ethereum address, and `tx_claim` to Alice. He also privately signs `tx_claim`, creating `claim_sig`.
4. Alice checks that the `tx_claim` is well-formed and locks her ETH in the swap contract, setting Bob's Ethereum address as the claimer.
5. Bob sees that the ETH was locked and confirms that the claimer address is correct. He then locks his XMR in the Monero address with public key `P_a + Y_b`, and the private key is `x_a + y`.
6. After `timeout_0` passes, Bob submits the signed `tx_claim` on-chain. This reveals `claim_sig`, which Alice combines with `claim_adaptor` to reveal the secret encryption key `y`. 
7. Alice claims the XMR in the acount `P_a + Y_b` as she now knows the private key, which is `x_a + y`.

#### 6.2.3 Refund path

1. Same as above.
2. Same as above. 
3. Same as above. Additionally, Alice prepares `tx_refund` that calls `refund`. She sends Bob an adaptor signature`refund_adaptor`  on `tx_refund` where the public encryption key is `P_a`. She also privately signs `tx_refund`, creating `refund_sig`. If Alice decides to call `refund`, `x_a` is revealed to Bob.
5. If Bob does not lock the XMR before `timeout_0`, Alice calls `refund`. 
6. If Bob locks his XMR, the step is the same as step 5 above.
7. If Bob locked his XMR, but was unable to claim the ETH before `timeout_1`, Alice will call `refund`. Then, he can combine `refund_sig` with `refund_adaptor` to reveal `x_a` and regain access to the XMR locked in `P_a + Y_b`.

### 6.3 Limitation

Currently, this protocol is not possible. This is because there is no way to check for an account nonce within an Ethereum smart contract. Since account nonce is part of the transactions `tx_claim` and `tx_refund`, it's possible for a party to create a transaction and adaptor with some nonce `nonce`, but then submit some transaction in-between sending the adaptor and calling the swap contract. Then, the contract will still accept the transaction, but the signature will be on a transaction with `nonce + 1`. Since this transaction is not the same as that signed by the adaptor, the counterparty will be unable to extract the secret encryption key.

## 7. Further work

### 7.1 Contractless protocol

Implementing an ETH-XMR atomic swap that does not rely on a smart contract deployed on-chain provides two benefits: firstly, reduced gas costs; secondly, enhanced privacy for both participants. 

However, a contractless protocol would actually work between any two blockchains, and not be limited to just ETH-XMR or even BTC-XMR. This protocol may be used between any two blockchains using ECDSA, Schnorr, or BLS signature schemes (TODO: check w/ VTS) and thus can be used as a base-level protocol for a cross-chain decentralized exchange (DEX).

### 7.2 Network-level privacy

In this section, I will briefly discuss privacy concerns related to the network implementation. 

Currently, the protocol requires at least 2 messages to be exchanged. These are the key exchange messages exchanged by the swap participants to initiate the swap. Assuming the participants are able to watch events occuring on the blockchain, no other messages necessarily need to be sent.

#### 7.2.1 Sender-recipient privacy

For a swap to initiate, the two parties must establish a connection between each other to exchange their swap keys. However, this reveals the IPs of each party to the counterparty. A party who wishes to monitor the network and determine who is requesting initiation of swaps, and with what amount, can easily do so without monetary recourse. They can simply spin up nodes that join the network and advertise swap offers and record who attempts to initiate with them, thereby determining IP addresses that want Monero. Alternatively, they can search for providers offering XMR, and thus find the IPs of users who reportedly own Monero. 

A simple solution is for users of the swap to use a VPN service. This will hide their real IP from other network participants. The drawback is that the user must trust the VPN provider not to publish their network activity.

Two potential solutions for this are onion routing or a mixnet. Onion routing allows network messages to be wrapped in successive layers of encryption, which must be unwrapped by each relaying node one at a time until the destination is reached. This hides the sender of the message from the relay nodes as well as the receipient, and hides the recipient's address from the sender. There exist implementations of a Tor transport for the lip2p library used by the current swap implementation, which may be integrated for increased privacy.

Alternatively, a mixnet can also be utilized. A mixnet provides greater privacy and resistance to metadata attacks than onion routing. Messages sent via onion routing may be correlated by timing and packet size. A mixnet prevents against this by "mixing" different packets together before sending them out from a relay node, as to obfuscate the size of a packet. As well, the relay node holds on to them for random intervals of time. This prevents against both timing attacks at the cost of greater latency. Using a mixnet for the networking implementation of the swap would be ideal, as latency is not critical; the swap users can simply set a sufficiently large timeout to negate the effects of latency. However, more research would need to be conducted into existing mixnet implementations that can be used. (TODO: look into some?)

#### 7.2.2 DHT privacy

The current atomic swap implementation uses a distributed hash table (DHT) for discovery of peers on the network, particular those who are providing swap offers. While not necessaily within the scope of an atomic swap protocol, which does not dictate how the participants meet, any implementation that uses a DHT will need to consider the privacy implications. However, a DHT is preferred for peer discovery over centralized alternatives such as a website, which may be censored or taken down by the hosting party. Additionally, by discovering peers through a centralized service, the website host becomes aware of every IP that visits the website, which would not be the case with a swap implementation that operates entirely through a peer-to-peer network.

DHTs are used to store and retrieve content in a decentralized, peer-to-peer network. Content is replicated between nodes, which can then be retrieved by a user by querying nodes in the network to "get closer" and eventually retrieve the contract they are searching for. DHTs are used in a blockchain context for peer discovery. A node in the network will advertise themselves in the DHT which can then be found by another party searching through the DHT. Similarly, they are used in the swap implementation to discover peers.

Most DHT implementations are not privacy-preserving and leak metadata about who is hosting what content, who is requesting it, and when. Similarly to the previous section, someone who wishes to extract metadata from the network can create a large number of nodes, wait until the DHT content gets replicated to them, then determine who is requesting that data. 

There has been research done into DHT implementations that preserve privacy of lookups, for example Octopus [4]. A swap implementation which aims to be as privacy-preserving as possible should use a privacy-preserving DHT implementation. (TODO: more research into impls)

### 7.3 Direct swaps for ERC20/ERC721 tokens



## 8. Conclusion

In summary, a protocol for trustless, peer-to-peer ETH-XMR atomic swaps has been described in this article. As well, limitations of the protocol in regards to privacy, griefing attacks, and gas costs has been discussed. A future protocol that is entirely contractless that uses adaptor signature, discrete logarithm equality proofs and verifiable timed signatures has been described at a high level. Future research and implementation work has been outlined regarding network privacy and direct Ethereum-based token swaps.

## Acknowledgements

Huge thank you to the Monero community for seeing the value in the project and funding this research.

## References
[1] Joel Gugger. Bitcoin–Monero Cross-chain Atomic Swap, 2020.  https://eprint.iacr.org/2020/1126.pdf
[2] Philipp Hoenisch and Lucas Soriano del Pino. Atomic Swaps between Bitcoin and Monero, 2021. https://arxiv.org/pdf/2101.12332.pdf
[3] Sarang Noether. Discrete logarithm equality across groups, 2018. https://www.getmonero.org/resources/research-lab/pubs/MRL-0010.pdf
[4] Qiyan Wang and Nikita Borisov. Octopus: A Secure and Anonymous DHT Lookup,  2012. https://arxiv.org/pdf/1203.2668.pdf
[5] Thyagarajan et al. Verifiable Timed Signatures Made Practical, 2020. https://eprint.iacr.org/2020/1563.pdf