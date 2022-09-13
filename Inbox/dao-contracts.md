# DAO DAO Contracts Design

The DAO DAO contracts are governance Lego blocks. They are designed to
be pluggable and upgradable. This enables highly customizable
governance configurations that are upgradable in the face of
vulnerabilities or bugs.

Each DAO is composed of at least three modules:

1. A [core
   module](https://github.com/DA0-DA0/dao-contracts/tree/cef33570e41c52b04acd692cdd527c836185fd13/contracts/cw-core). This
   handles execution of messages on behalf of the DAO and holds the
   DAO's treasury.
2. A voting power module. This module is responsible for determining
   the voting power of members. For example, it may be based on how
   many governance tokens you [have
   staked](https://github.com/DA0-DA0/dao-contracts/tree/cef33570e41c52b04acd692cdd527c836185fd13/contracts/cw20-staked-balance-voting),
   or
   [multisig-style](https://github.com/DA0-DA0/dao-contracts/tree/cef33570e41c52b04acd692cdd527c836185fd13/contracts/cw4-voting)
   voting weights.
3. At least one proposal module. Proposal modules handle the lifecycle
   of proposals. Different modules may support, for example,
   [single](https://github.com/DA0-DA0/dao-contracts/tree/cef33570e41c52b04acd692cdd527c836185fd13/contracts/cw-proposal-single)
   or [multiple
   choice](https://github.com/DA0-DA0/dao-contracts/tree/cef33570e41c52b04acd692cdd527c836185fd13/contracts/cw-proposal-multiple)
   voting. A DAO may register more than one proposal module to support
   any number of proposal types. Proposal modules are very powerful
   and can do [way
   more](https://github.com/DA0-DA0/dao-contracts/discussions/373)
   than just manage proposals.

All module types have well defined interfaces. This makes it possible
for DAOs to implement custom governance systems where they want and
use DAO DAO's [audited
modules](https://github.com/oak-security/audit-reports/tree/master/DAO%20DAO)
elsewhere. For example, Neta DAO uses a [custom proposal
module](https://github.com/DA0-DA0/dao-contracts/compare/DA0-DA0:cef3357...onewhiskeypls:e460259)
that allows a smaller subDAO counsel to veto proposals.

Speaking of subDAOs, in addition to those modules a DAO may have an
admin set. The admin of a DAO may execute messages on the DAO and,
thus, has complete administrative control of the DAO. This enables DAO
DAO subDAOs where a parent DAO may spin up and oversee the operations
of a more lean, task based subDAO.

Here's a high level diagram of how those all fit together:

```
                                                +--------------+
                                                |              |
                                              /-+  proposal    |
   +------------+       +-------------+  /----  |              |
   |            |       |             +--       +--------------+
   |  voting    +-------+  core       |
   |            |       |             +--       +--------------+
   |            |       |             |  \----  |              |
   +------------+       +---/---------+       \-+ proposal     |
                           /                    |              |
                          /                     +--------------+
                         /
                   +----/------+
                   |           |
                   |  item     |
                   |           |
                   +-----------+
```

## The core contract

```
     (2) validate is registered proposal contract
     +-----------------+
     |                 |
     |                 |  (1) execute message hook
     |    core         | <------------
     |                 |
     |                 |
     +-----------------+
             |
             | (3) execute the message
             v
```

The core contract executes messages on behalf of the proposal
modules. To do this it has an `ExecuteProposalHook { msgs }`
method. This method may only be called by proposal modules that have
been registered with the core contract. Controller modules may
call this method when they have messages they would like to see
executed. For example, this may occur when a proposal module has
passed a proposal.

The core contract may also route queries between proposal modules and
its voting module about voting power. To this end it implements the
same interface for queries that voting modules must implement.
Specifically, it implements a `VotingPowerAtHeight { addr, height }`
and `TotalPowerAtHeight { height }` query. These queries will be
proxied to the core module's voting module and that response returned.

At all times the core contract must have one voting module associated
with it and at least one proposal module associated with it.

The core contract may add and remove these modules at will by
calling the `UpdateVotingModule` and `UpdateGovernanceModules` methods
on itself. In practice this occurs by proposal modules sending a
execute hook that will cause the core contract to call that method
on itself.

### subDAOs

To better support subDAOs the core contract may also be instantiated
with an admin. This admin has the ability to execute messages on the
core module outside of the regular governance flow. The current admin
may also nominate new admins or remove themselves. Having this admin
functionality allows parent DAOs to exercise some control over
subDAOs.

The process for assigning a new admin has two steps:

1. The current admin nominates a new admin.
2. The new admin accepts the nomination and becomes the admin.

These are accomplished via the `NominateAdmin` and
`AcceptAdminNomination` messages. The current admin may, at any time,
withdraw their nomination by executing a `WithdrawAdminNomination`
message. In order to nominate a new admin any outstanding nomination
must be withdrawn.

The admin is not able to execute messages on a paused subDAO.  Future
versions of the core contract may support this. Please let us know if
you have a use case for this.

## Items

```
   +---------------+                  +----------------+
   | core          |                  |                |
   |               |                  |special purpose |
   |               |              /-->|contract        |
   |           +-----+    /-------    |                |
   |           | key |----            +----------------+
   +-----------+-----+
```

Items are key value pairs stored in the core contract. Their key is a
string and their value is a string. One may use these to store
information like the social media addresses of a DAO, or the contract
address of a related smart contract.

The core contract may add and remove these items by calling the
`SetItem` and `RemoveItem` methods on itself. In practice this occurs
by proposal modules sending a execute hook that will cause the
core contract to call that method on itself.

## The voting contract

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

## Proposal modules

```
 +----------------+                    +--------------+
 | core           |   execution hooks  | proposal     |
 |                | <------------------|              |
 |                |                    |              |
 |                |                    |              |
 +----------------+                    +--------------+
                                               ^
                                               | proposal information
                                               |
```

Proposal contracts are extremely flexible and don't have any
particular interface they must implement. They only need to be aware
of how to execute the proposal hook on the core contract.

It is imagined that proposal contracts will mostly implement some
kind of proposal and voting system. For example, they might support
ranked choice voting, multiple choice voting, or quadratic voting. In
practice, this is not enforced and a core contract may register any
proposal contract it would like.

`cw-proposal-single` is the standard proposal contract and allows for
single choice (yes or no) voting. Proposal contracts can manage
multiple proposals so a new one is not created for each proposal. In
the case of `cw-proposal-single` the contract implements a variety of
queries to help the frontend display proposal information. Future
proposal module authors may consider implementing a similar interface.

## Frontend integration

```
                   (2) what kind of module are you?
   +-------------+ -----------------> +----------+
   | frontend    |                    | module   |
   |             |                    |          |
   |             | -----------------> +----------+
   +-------------+ (3) module specific queries
          |
          |
          | (1) what modules do you have?
          |
          v
    +-----------+
    |core       |
    |           |
    |           |
    +-----------+
```

All modules in this system must implement an `Info {}` query which
return the contract's name and its version. To render a module the
frontend will then query a the core contract for its modules and then
query those modules for their info.
