# angrymouse approach
## Introduction
Since bitcoin's invention, many blockchains were created with decentralization in mind. And it works: You really can install full chain node and have access to your blockchain in decentralized way. 

But operating full blockchain node requires relatively good technical knowledge, open ports, big SSD drive and CPU (sometimes even GPU to make it faster) powers.

At the moment of writing (09.08.2022) Bitcoin's full node requires around 420GB of disk space, Ethereum's full node requires around 900 GB of space (non-archival, archival requires 11.7 TB), and there's much more examples of growing chain sizes (while even if you have disk, operating node requires strong CPU and fast internet connection to sync and support full node), days of sync and great internet connection without payment for traffic consumed.

Because of complexity described above, most users use "Nodes-as-a-service" services like [Infura](https://infura.io/).

These services are not requesting your wallet key, so you still have control over your funds. However, centralization still stays as main lack of such services.

Existence and adoption of such services makes blockchains and platforms based on it not censorship-resistant and centralized in some way.

Tornado Cash, ethereum's crypto mixer that pretended to be decentralized, was blocked in several ways, one of these ways and one of main ways was blocking interactions with Tornado Cash using Infura API endpoint.

From this event we understood where should we decentralize our dApps, and interfaces for chain queries are one of the most important points.

## Voting Power
## The [[dao-contracts]] approach
```
      +------------------+
      |                  |
      |                  |   voting power queries
      |                  | <------------
      |   voting         |
      |   (voting power  |
      |    management)   |
      |                  |
      |                  |
      +------------------+
```

The voting contract manages the voting power of governance
members. For example, for a multisig style voting system this contract
may allow the DAO to add and remove members and manage that member
list. For a staking style voting system this module may allow
addresses to stake tokens and report information about the number of
tokens that they have staked.

Voting contracts must implement a `VotingPowerAtHeight` and
`TotalPowerAtHeight` query which controller modules will use to
determine the weight of an addresses vote on a proposal. For voting
modules that do voting based on a token they must also implement a
`TokenContract` query which will return the address of the token being
used. Proposal modules that support proposal deposits may query this
information to determine what token to accept for deposits.

In practice, there are no hard requirements on what queries that a
core module actually implements as sufficiently custom proposal
modules may query the module directly and may only be compatible with
specific voting systems.
## Solution

### Lite nodes

Lite node is blockchain node connects to blockchain's p2p swarm, acts like full chain node, but not contains and operates full chain. Such nodes usually rely on other full nodes in p2p swarm. [SPV](https://en.bitcoinwiki.org/wiki/Simplified_Payment_Verification) is one of ways of how lite nodes should work.
But SPV haves few lacks:
- It is very complex to use it with Proof of Stake blockchains.
- Lite nodes require healthy altruistic full nodes in network.
- Not suitable for smart contracts platforms, as there's big contract state needed to confirm.

### Dedicated network
Solution can be found in dedicated Proof of Stake network for resolving queries.

When we use PoS as our consensus algorithm, let's assume that we trust majority of validators in this network, so why not make them sign query replies for you?

We only need lite node to store list of voting powers of our validators to work well.

#### Self-migrated list of voting powers
We can make upgrades of voting powers knowing only previous voting powers.

To do it, we assume that there's no big (>67%) changes in voting power between 2 different voting power states (We assume it in every PoS chain, if there's big change of voting powers, old validators can just reject signing delegation transaction and not let move themselves out of control).

If we take this assumption, we can make old validators sign new list of voting powers and serve it.

Scheme below describes how self-migrated voting power lists can work.

![self-updating-voting-powers-01](https://user-images.githubusercontent.com/40735471/189504918-d3b5866a-10e3-4307-917b-b3983557ebc3.png)

#### Blind transactions

If we look precisely, then we can understand that we don't need any info about activity on chain to submit stateless transactions (transfers in state-based networks and similar, in UTXO model it not works because you need to lookup which UTXO can you use).

So we can avoid using centralized API endpoints for broadcasting transactions, just form transaction, sign and broadcast directly to p2p swam.

#### Safe and decentralized queries with only small piece of knowledge about chain

Now, when we understand how to keep voting powers always updated, we have ultimate method to verify data about chain without storing chain itself: We can just ask validators to sign reply to query and give it to you. Reply that will get 51%+ of voting power can be considered as truth.

##### Incentives

But if you don't incentivize validators to sign reply for your query, they most likely won't do it.

Validators have incentive to add your transactions to blocks: They get mining fee.

Queries load validator nodes not less if not more (nodes need to go through indexes and databases to form right reply for you). 

So we need to add mining fee for queries too, to prevent abuse of overloading validator nodes with queries and reward validators who reply with right data.

#### Pledge system

As we already know, we can make stateless, "blind" transactions knowing absolutely nothing about chain state.

It allows us to incentivize validators to reply on your queries.

It involves making query into 2-layer structure/protocol that contains 2 transactions in it:
1. Pledge deposit and query info
2. Finalizing transaction. Rewarding of validators who replied and punishment of ones who replied with misleading info, claiming 40% of pledge back

So way of our query will look like this:
- Lite node needs some info about chain, it forms query, selects pledge amount that it thinks most validators will accept, selects mining fee for pledge transaction and submits it in blind way.
- Lite node provided way (in pledge transaction) how to deliver reply to it, and now it listens for answers of validators. 
- Validators receive pledge transaction, they calculate, sign and broadcast back reply they consider valid.
- As soon as lite node received enough replies to select reply that got 51% of voting power, it resolves this reply to app that requested it, but still waits 1 minute to collect as much replies as possible to go to next stage.
- Now lite node collected all replies from all validators that are active and reply to queries, so it can make finializing transaction that reports about who replied, who didn't reply, and who replied with misleading info.
- After finalizing transaction is confirmed on chain, validators claim part of pledge based on their part of voting powers, 40% of pledge is returned to lite node's wallet (it financially incentivizes lite node to submit finalizing transaction).
 - Validators who didn't reply aren't getting any punishment (we can't verify that lite node is not lying about that it didn't get reply, so no punishment can be applied). However they don't get allocated part of pledge neither: It gets burned. 
 - To disincentivize lite nodes to report false info and don't give part of pledge to validators, lite node's part of pledge (that returns to node's wallet after finalizing transaction is done) gets locked for 10 days (not fully, but only part that is proportional to part of pledge allocated to validator that didn't get his part because node reported that validator didn't answer).
 - Validators who replied with misleading info get jailed and slashed (other validators can easily verify that validator did reply wrong, as validators sign their replies with their consensus keys)

During this way, we fetched data from chain without connecting to centralized API endpoints and without storing and processing chain itself.

Nik Rykov, nik@hns.is, 09.12.2022