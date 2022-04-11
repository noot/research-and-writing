# GRANDPA finality gadget

from the polkadot wiki:
https://wiki.polkadot.network/docs/learn-consensus/#hybrid-consensus
> There are two protocols we use when we talk about the consensus protocol of Polkadot, GRANDPA and BABE (Blind Assignment for Blockchain Extension). We talk about both of these because Polkadot uses what is known as hybrid consensus. Hybrid consensus splits up the finality gadget from the block production mechanism.
> This is a way of getting the benefits of probabilistic finality (the ability to always produce new blocks) and provable finality (having a universal agreement on the canonical chain with no chance for reversion) in Polkadot. It also avoids the corresponding drawbacks of each mechanism (the chance of unknowingly following the wrong fork in probabilistic finality, and a chance for "stalling" - not being able to produce new blocks - in provable finality). By combining these two mechanisms, Polkadot allows for blocks to be rapidly produced, and the slower finality mechanism to run in a separate process to finalize blocks without risking slower transaction processing or stalling.


overview:
- GRANDPA (GHOST-based Recursive ANcestor Deriving Prefix Agreement) is the finality gadget for Polkadot
- it is the algorithm that authority nodes run to finalize blocks
- BABE for building blocks runs completely separately, they do not depend on each other in any way
- assumption: GRANDPA works in a partially synchronous network as long as >=2/3 of nodes are honest
- partially synchronous: network messages arrive within some fixed window of time

> Note: GRANDPA finalizes *chains*, not *blocks*. So, while the votes are for a specific block, the vote is implicitly for every ancestor of that block as well. 

GRANDPA algorithm
- GRANDPA is executed in "rounds" (currently ~2s on mainnet)
- every round, authority nodes participate in the algorithm by voting on what block they think should be finalized
- at the end of each round, a block is finalized
- nwely finalized block's number is always >=(previously finalized block's number)
- a round can take longer than 2s (or whatever the expected time is) if not enough nodes participate (ie. more than 1/3 of nodes are offline or there is some network partition)
- however once >=2/3 votes are received in a round, it can be finalized

within each round, there are 2 sub-rounds:
- prevote
- precommit

the prevote round is the initial round where nodes broadcast what block they think is best.
in the precommit round, nodes adjust their vote based on the results of the prevote subround.

each round, every authority node on the network runs the following algorithm:
1. if the node is "primary", it broadcasts a `primaryProposal` message which is essentially a prevote for its best block. the primary is determined in round-robin fashion (index = round % |authority set|)
2. nodes determine and broadcast their prevote. if a `primaryProposal` message was recevied for some block, and that block known by the node and is higher than the previously finalized block, then their prevote is for that same block. otherwise, the prevote is for their own best block.
3. wait for prevotes, continue to next stage when # of prevotes received (from different authorities) is >=2/3 |authority set|
4. nodes determine and broadcast their precommit. the precommit is for the *pre-voted block* which is the highest block that received >=2/3|authority set| prevotes. since votes are for *chains*, not *blocks*, this block may not be one that was directly voted for but may be an ancestor of directly-voted-for blocks. 
5. wait for precommits, when >=2/3|authority set| precommits are received for the **best final candidate** then that block is finalized and a **commit message** is broadcast to the network. the best final candidate (bfc) is determined as follows:
  >  i. determine all blocks with height >= (height of pre-voted block) with >=2/3|authority set| precommit votes
    ii. return the block in this set with the highest number
    
Note: equivocations
- a vote is "equivocatory" if it conflicts with a vote already sent out by that authority
- ie. if an authority sends out 2 different votes in the same subround that are for blocks on different chains
- these votes are added to an equivocations set
- the contents of the votes are no longer used, but the total number of equivocators is still used in the block vote count calculation
- equivocations are very bad ie. if a node equivocates it is obviously malicious and will be slashed
    
Note: how are votes for a block counted? 
- vote messages passed by grandpa contains (block hash, block number)
- to calculate the # of votes for some block, you need to take into account votes for all its descendants as well as equivocatory votes.
> total_votes(B) = (number of votes for B or some descendant of B) + (number of equivocating authorities)
    

staking vs GRANDPA
- staking or proof of stake is a separate mechanism to GRANDPA
- polkadot is proof of stake in that nodes must stake to become authorities and they will be slashed if they perform some invalid action
- however, staking is simply how authorities get added to the active set, stake does not have a role in how the algorithm plays out
- grandpa has the ability to have weighted authorities, where instead of >=2/3|authority set| for a block to be finalized it's >=2/3(total stake). however this is not used on mainnet and every authority has the same weight


additional reading:
https://medium.com/polkadot-network/polkadot-consensus-part-2-grandpa-fb1963ef6c70
https://medium.com/polkadot-network/polkadot-proof-of-concept-3-a-better-consensus-algorithm-e81c380a2372
https://medium.com/polkadot-network/polkadot-consensus-part-4-security-eb3180b6d7e4