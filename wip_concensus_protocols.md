# Consensus Protocols
#### October 2021

#### Contents

1. Goal of consensus protocols
2. Overview of common concepts
    a. block producer selection and block production
    b. chain selection
    c. finality
3. Algorithms
    a. eth2
    b. Polkadot (BABE+GRANDPA)
    c. Tendermint
    d. Ouroboros
    
### 1. Goal of consensus protocols

- A blockchain consists of a decentralized p2p network in which each node has a state
- Additionally, new blocks (or states) regularly get proposed to the network
- The goal of a consensus protocol is:
    -  for every correctly-operating node on the network to agree on the same state, as long as there is a suffient number of correctly-operating nodes (**safety**)
    - for the algorithm to continue to make progress (ie. continue to create new blocks) (**liveness**)

### 2. Overview of common concepts
#### 2a. Block producer selection

- validator (authority) node: a node that participates in block production
- in general, there are two types of block producer sets: "infinite" and finite
    - an infinite block producer set means that any node which runs the block production algorithm may produce a valid block; there is no fixed list of validators
    - a finite block producer set means that a list of authorized nodes is stored on-chain; some criteria must be met to get included in this set

How are block producers selected?
- Proof-of-Work: infinite block producer set; any miner who is able to overcome the block  difficulty can produce a valid block
- Proof-of-Stake: block producers must stake a certain amount of value before being able to produce a valid block (can be infinite or finite set, depending on algorithm)
- Proof-of-Authority: a fixed authority set is provided

Block production: 
- exact mechanism depends on algorithm
- in general, a validator node will  either continuously try to build or mine a block (if there are transactions to be included); or, the validator will wait until its designated time to build a block
- once block is built, it is gossiped to the network, after which the chain selection algorithm will determine whether is becomes part of the canonical chain or nott

#### 2b. Chain selection

Each node on the network, assuming the network is healthy, should see the same view of all the potential chains on the network (TODO: add diagram)
- the chain selection algorithm deals with the selection of the best or canonical chain
- if each node has the same view of the chains, each correctly-operating node should choose the same chain
- most well known is GHOST for Bitcoin; essentially "heaviest" chain is selected (TODO: add diagram)

#### 2c. Finality
Probabilistic vs. provable (deterministic) finality
- finality: when a block is "finalized", it will never be reverted; ie. it will be part of the canonical chain forever
- eg. when is my transaction "confirmed"?
- provable finality: a cryptographic guarantee that a block will never be reverted
- probablisitic finality: the probability of a block being reverted gets smaller as its number of descendents grows; can approach 100% but is never 100%
    - this is why chains like Bitcoin require a certain number of "confirmations" or additional blocks past the block in which the tx was included

provable finality requires a finite validator set and >2/3 correctly-operating validators to function (BFT consensus)
- BFT consensus has voting rounds where validators vote on what block to finalize
- finite validator set is needed since network overhead is large
- examples: Casper FFG, GRANDPA, Tendermint

probabilistic finality (Nakamoto consensus) requires only >50% of validators/miners to be correctly-operating
- examples: Bitcoin, ethash, Ouroboros


### 3. Algorithms
#### 3a. eth2
- Proof-of-Stake
- validators take turns proposing and voting on blocks
- weight of validator's vote is proportional to stake

#### 3b. Polkadot

#### 3c. Tendermint

#### 3d. Ouroboros


### Resources
https://docs.ethhub.io/ethereum-roadmap/ethereum-2.0/proof-of-stake/