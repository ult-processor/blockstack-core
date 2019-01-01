# Abstract

This proposal describes a mechanism for single-leader election using
_proof-of-burn mining_.  The Stacks blockchain grows in tandem with an
underlying "burn blockchain" whose cryptocurrency tokens are destroyed in order to produce a
Stacks block.  Proof-of-burn mining is concerned with deciding
which Stacks block miner (called _leader_ in this text) is elected for
producing the next block, as well as deciding how to resolve
conflicting transaction histories.

# Introduction

Blockstack's first-generation blockchain operates in such a way that each transaction is
in 1-to-1 correspondence with a Bitcoin transaction.  The reason for doing this
is to ensure that the difficulty of reorganizing Blockstack's blockchain is just
as hard as reorganizing Bitcoin's blockchain -- a [lesson
learned](https://www.usenix.org/node/196209) from when the system was originally
built on Namecoin.

This SIP describes the proof-of-burn consensus algorithm in
Blockstack's second-generation blockchain (the _Stacks
blockchain_).  The Stacks blockchain makes the following improvements over the
first-generation blockchain:

## 1. High validation throughput

The number of Stacks transactions processed is decoupled from the
transaction processing rate of the underlying burn chain (Bitcoin).  Before, each Stacks
transaction was coupled to a single Bitcoin transaction.  In the Stacks
blockchain, an _entire block_ of Blockstack transactions corresponds to a
Bitcoin transaction.  This significantly improves cost/byte ratio for processing
Blockstack transactions, thereby effectively increasing its throughput.

## 2. Low-latency block inclusion

Users of the first version of the Stacks blockchain encounter high latencies for
on-chain transactions -- in particular, they must wait for an equivalent
transaction on the burn chain to be included in a block.  This can take 
minutes to hours.

The Stacks blockchain adopts a _block streaming_ model whereby each leader can
adaptively select and package transactions into their block as they arrive in the
mempool.  This ensures users learn when a transaction is included in a block
on the order of _seconds_.

## 3. An open leadership set

The Stacks blockchain uses proof-of-burn mining to decide who appends the next
block.  The protocol (described in this SIP) ensures that anyone can become
a leader, and no coordination amongst leaders is required to produce a block.
This preserves the open-leadership property from the existing blockchain (where
Blockstack blockchain miners were also Bitcoin miners), but realizes it through
an entirely different mechanism that enables the properties listed here.

## 4. Participation without mining hardware

Producing a block in the Stacks blockchain takes negligeable energy on top of
the burn blockchain.  Would-be miners append blocks by _burning_ an existing cryptocurrency
by rendering it unspendable.  The rate at which the cryptocurrency is destroyed
is what drives block production in the Stacks blockchain.  As such, anyone who
can acquire the burn cryptocurrency (e.g. Bitcoin) can participate in mining,
_even if they can only afford a minimal amount_.

## 5. Fair mining pools

Related to the point above, it is difficult to participate in block mining in
a proof-of-work blockchain.  This is because a would-be miner needs to
lock up a huge initial amount of capital in dedicated mining hardware,
and a miner receives few or no rewards for blocks that are not incorporated
into the main chain.  Joining a mining pool is a risky alternative,
because the pool operator can simply abscond with the block reward or
dole out block rewards in an unfair way.

The Stacks blockchain addresses this problem by providing
a provably-fair way to mine blocks in a pool.  To
implement mining pools, users aggregate their individually-small burns
to produce a large burn, which in turn gives them all a
non-negligeable chance to mine a block.  The leader election protocol
is aware of these burns, and rewards users proportional to their
contributions _without the need for a pool operator_.  This both
lowers the barrier to entry in participating in mining, and removes
the risk of operating in traditional mining pools.

In addition to helping lots of small burners mine blocks, fair mining pools
are also used to give different block leaders a way to hedge their bets on their
chain tips:  they can burn some cryptocurrency to competing chain tips and
receive some of the reward if their preferred chain tip loses.  This is
important because it gives frequent leaders a way to reduce the variance of
their block rewards.

## 6. Ability to migrate to a separate burn chain in the future

A key lesson learned in the design of the first-generation Stacks blockchain is that
the network must be portable, in order to survive systemic failures such as peer
network collapse, 51% attacks, and merge miners dominating the hash power.
The proof-of-burn mining system will preserve this feature,
so the underlying burn chain can be "swapped out" at a later date if need be.

## Assumptions

Given the design goals, the Stacks leader election protocol makes the following
assumptions:

* Deep forks in the burn chain are exponentially rarer as a function
  of their length.

* Deep forks in the burn chain occur for reasons unrelated to the
  Stacks protocol execution.  That is, miners do not attempt to
  manipulate the execution of the Stacks protocol by reorganizing the
  burn chain (but to be clear, burn chain miners may participate in the Stacks
chain as well).

* Burn chain miners do not censor all Stacks transactions (i.e. liveness is
  possible), but may censor some of them.  In particular, Stacks transactions
on the burn chain will be mined if they pay a sufficiently high transaction
fee.

* At least 2/3 of the Stacks leader candidates, measured by burn
  weight, are correct and honest.  If there is a _selfish mining_ coalition,
then we assume that 3/4 of the Stacks leader candidates are honest (measured
again by burn weight) and that honestly-produced Stacks blocks propagate to the honest 
coalition at least as quickly as burn blocks (i.e. all honest peers receive the
latest honest Stacks block data within one epoch of it being produced).

# Protocol overview

Like existing blockchains, the Stacks blockchain encodes a cryptocurrency and
rules for spending its tokens.  Like existing cryptocurrencies, the Stacks
blockchain introduces new tokens into circulation each time a new block is
produced (a _block reward_).  This encourages peers to participate in gathering transactions and
creating new blocks.  Peers that do so so called _leaders_ of the blocks that
they produce (analogous to "miners" in existing cryptocurrencies).

Blocks are made of one or more _transactions_, which encode valid state
transitions in each correct peer.  Users create and broadcast transactions to the peer
network in order to (among other things) spend the tokens they own.  The current
leader packages transactions into a single block during its epoch -- in this
way, a block represents all transactions processed during one epoch in the
Stacks chain.

Like existing cryptocurrencies, users compete with one another for _space_ 
in the underlying blockchain's peers for storing their transactions.  This competition is
realized through transaction fees -- users include an extra Stacks token payment in
their transactions to encourage leaders to incorporate their transactions
first.  Leaders receive the transaction fees of the transactions they
package into their blocks _in addition_ to the tokens minted by producing them.

Blocks are produced in the Stacks blockchain in cadence with the underlying burn
chain.  Each time the burn chain network produces a block, at most one Stacks block
will be produced.  In doing so, the burn chain acts as a decentralized
rate-limiter for creating Stacks blocks, thereby preventing DDoS attacks on its
peer network.  Each block discovery on the burn chain triggers a new _epoch_ in the Stacks
blockchain, whereby a new leader is elected to produce the next Stacks block.

With the exception of a designated _genesis block_, each block in the Stacks
blockchain has exactly one "parent" block.  This parent relationship is a
partial ordering of blocks, where concurrent blocks (and their descendents)
are _forks_.

If the leader produces a block, it must have an already-accepted block as its
parent.  A block is "accepted" if it has been successfully
processed by all correct peers and exists in at least one Stacks blockchain fork.
The genesis block is accepted on all forks.

Unlike most existing blockchains, Stacks blocks are not produced atomically.
Instead, when a leader is elected, the leader may dynamically package
transactions into a sequence of _microblocks_ as they are received from users.
Logically speaking, the leader produces one block; it just does not need to
commit to all the data it will broadcast when its tenure begins.  This strategy
was first described in the [Bitcoin-NG](https://www.usenix.org/node/194907) system,
and is used in the Stacks blockchain with some modifications -- in particular, a
leader may commit to _some_ transactions that _must_ be broadcast during its
tenure, and may opportunistically stream additional transactions in microblocks.

## Novel properties enabled by proof-of-burn mining

Each Stacks block is anchored to the burn chain by
way of a cryptographic hash.  That is, the burn chain's canonical transaction
history contains the hashes of all Stacks blocks ever produced -- even ones that
were not incorporated into any fork of the Stacks blockchain.  This gives the
Stacks blockchain three properties that existing blockchains do not possess:

* **Global knowledge of time** -- Stacks blockchain peers each perceive the passage of time
  consistently by measuring the growth of the underlying burn chain.  In
particular, all correct Stacks peers that have same view of the burn chain can
determine the same total order of Stacks blocks and leader epochs.  Existing blockchains do not
have this, and their peers do not necessarily agree on the times when blocks were produced.

* **Global knowledge of blocks** -- Each correct Stacks peer with the same
  view of the burn chain will also have the same view of the set of Stacks
blocks that *exist*.  Existing blockchains do not have this, but instead
rely on a well-connected peer network to gossip all blocks.

* **Global knowledge of burns** -- Each correct Stacks peer with the same view
  of the burn chain will know how much cumulative cryptocurrency was destroyed
in order to produce the blockchain (as well as when it was destroyed).  Existing
blockchains do not have this -- a private fork can coexist with all public
forks and be released at its creators' discression (often with harmful effects
on the peer network).

The Stacks blockchain leverages these properties to implement three key features:

* **Mitigate block-withholding attacks**:  Like all single-leader blockchains,
  the Stacks blockchain allows the existence of multiple blockchain forks.
These can arise whenever a leader is selected but does not produce a block, or
produces a block that is concurrent with another block.  The
design of the Stacks blockchain leverages the fact that all _attempts_ to produce
a block are known to all leaders in advance in order to detect and mitigate
block-withholding attacks, including selfish mining.  It does not prevent these
attacks, but it makes them easier to detect and offers peers more tools to deal
with them than are available in existing systems.

* **Ancilliary burns enhance chain quality**:  In the Stacks blockchain, peers can enhance
  their preferred fork's chain quality by _contributing_ burns to their
preferred chain tip.  This in turn helps ensure chain liveness -- small-time
burners (e.g. typical users) can help honest leaders that commit to
the "best" chain tip get elected, and punish dishonest
leaders that withhold blocks or build off of other chain tips.
Users leverage this property to construct _fair burn pools_, where users can
collectively burn to select a chain tip to build off of and receive a proportional share of the
block reward without needing to rely on any trusted middlemen to do so.

* **Ancilliary burning to hedge bets**:  Because anyone can burn in favor of any chain tip,
leaders can hedge their bets on their preferred chain tips by distributing
their burns across _all_ competing chain tips.  Both fair burn pools and burning
over a distribution of chain tips are possible only because all peers have 
knowledge of all existing chain tips and the burns behind them.

# Leader Election

The Stacks blockchain makes progress by selecting successive leaders to produce
blocks.  It does so via burn mining, whereby would-be leaders submit their candidacies
to be leaders by burning an existing cryptocurrency.  A leader is selected to produce a
block based on two things:

* the amount of cryptocurrency burned relative to the other candidates
* an unbiased source of randomness

A new leader is selected whenever the burn chain produces a new block -- the
arrival of a burn chain block triggers a leader election, and terminates the
current leader's tenure.

The basic structure for leader election through burn-mining is that
for some Stacks block _N_, the leader is selected via some function of
that leader's total cryptocurrency burnt in a previous block _N'_ on the
underlying burn chain.  In such a system, if a candidate _Alice_ wishes to be a leader of a
Stacks block, she issues a burn transaction in the underlying burn
chain. The network then uses cryptographic sortition to choose a
leader in a verifiably random process, weighted by burn amounts in prior blocks.
The block in which this burn transaction is broadcasted is known
as the "election block" for Stacks block _N_.

Anyone can submit their candidacy as a leader by issuing a burn transaction on
the underlying burn chain, and have a non-zero chance of being selected by the
network as the leader of a future block.

## Committing to a chain tip

The existence of multiple chain tips is a direct consequence of the
single-leader design of the Stacks blockchain.
Because anyone can become a leader, this means that even misbehaving
leaders can be selected.  If a leader crashes before it can propagate
its block data, or if it produces an invalid block, then no block will
be appended to the leader's selected chain tip during its epoch.  Also, if a
leader is operating off of stale data, then the leader _may_ produce a
block whose parent is not the latest block on the "best" fork, in which case
the "best" fork does not grow during its epoch. These
kinds of failures must be tolerated by the Stacks blockchain.

A consequence of tolerating these failures is that the Stacks blockchain 
may have multiple competing forks; one of which is considered the canonical fork
with the "best" chain tip.  However, a well-designed
blockchain encourages leaders to identify the "best" fork and append blocks to
it by requiring them to _irrevocably commit_ to the
chain tip they will build on for their epoch.
This commitment must be tied to an expenditure of some
non-free resource, like energy, storage, bandwidth, or (in this blockchain's case) an
existing cryptocurrency.  The intuition is that if the leader does _not_ build on
the "best" fork, then it commits to and loses that resource at a loss.

Committing to a _chain tip_, but not necessarily new data,
is used to encourage both safety and liveness in other blockchains
today. For example, in Bitcoin, miners search for the inverse of a hash that
contains the current chain tip. If they fail to win that block, they
have wasted that energy. As they must attempt deeper and deeper forks
to initiate a double-spend attacks, producing a competing fork becomes an exponentially-increasing energy
expenditure.  The only way for the leader to recoup their losses is for the
fork they work on to be considered by the rest of the network as the "best" fork
(i.e. the one where the tokens they minted are spendable).  While this does
not _guarantee_ liveness or safety, penalizing leaders that do not append blocks to the 
"best" chain tip while rewarding leaders that do so provides a strong economic
incentive for leaders to build and append new blocks to the "best" fork
(liveness) and to _not_ attempt to build an alternative fork that reverts
previously-committed blocks (safety).

It is important that the Stacks blockchain offers the same encouragement.
In particular, the ability for leaders to intentionally orphan blocks in order
to initiate double-spend attacks at a profit is an undesirable safety violation,
and leaders that do so must be penalized.  This property is enforced by
making a leader announce their chain tip commitment _before they know if their
blocks are included_ -- they can only receive Stacks tokens if the block for
which they burned is accepted into the "best" fork.

## Election Protocol

To encourage safety and liveness when appending to the blockchain, the leader
election protocol requires leaders to burn cryptocurrency before they know
whether or not they will be selected.  To achieve this, the protocol for electing a leader
runs in three steps.  Each leader candidate submits two transactions to the burn chain -- one to register
their public key used for the election, and one to commit to their burn amount and chain tip.
Once these transactions confirm, a leader is selected and the leader can
append and propagate block data.

Block selection is driven by a _verifiable random function_ (VRF).  Leaders burn to
register their VRF proving keys, and later attempt to append a block by generating a
VRF proof over their preferred chain tip's _seed_ -- an unbiased random string
the leader learns after their burn is committed.  The resulting proof is used to
select the next block through cryptographic sortition, as well as the next seed.

The protocol is designed such that a leader can observe _only_ the burn-chain
data and determine the set of all Stacks blockchain forks that can plausibly
exist.  The on-chain data gives all peers enough data to identify all plausible
chain tips, and to reconstruct the proposed block parent relationships and 
block VRF seeds.  The on-chain data does _not_ indicate whether or not a block or a seed is
valid, however.

### Step 1: Register key

In the first step of the protocol, each leader candidate registers itself for a
future election by sending a _key transaction_. In this transaction, the leader
commits to the public proving key that will be used by the leader candidate to
generate the next seed for the chain tip they will build off of.

The key transactions must be sufficiently confirmed on the burn chain
before the leader can commit to a chain tip in the next step.  For example, the
leader may need to wait for 10 epochs before it can begin committing to a chain
tip.  The exact number will be protocol-defined.

The key transaction has a short lifetime in the protocol.  It must be consumed
by a subsequent commitment transaction within a small number of epochs
before it expires.  Once a key transaction is spent or expires,
the public proving key _cannot be used again_.

### Step 2: Burn & Commit

Once a leader's key transaction is confirmed, the leader will be a candidate for election
for a subsequent burn block in which it must send a _commitment transaction_.
This transaction both burns the leader's cryptocurrency
(proof-of-burn) and registers the leader's preferred chain tip and new VRF seed
for selection in the cryptographic sortition.

This transaction commits to the following information:

* the amount of cryptocurrency destroyed to produce the block
* the chain tip that the block will be appended to
* the proving key that will have been used to generate the block's seed
* the new VRF seed if this leader is chosen
* any transaction data that the leader _promises_ to include in their block (see
  "Operation as a leader").

The seed value is the cryptographic hash of the chain tip's seed (on the burn chain)
and this block's VRF proof generated with the leader's proving key.  The VRF proof
is stored in the Stacks block header off-chain, but committed to on-chain.

The leader has a 1-epoch window of time in which to generate a commitment
transaction that matches its key transaction (i.e. if the key transaction is
included at height _H_, then the commitment must be included at height _H+K_ for
fixed _K_).  This is because leaders cannot be allowed to have a choice as to
which seed they will build off of; otherwise they might be able to influence the
sortition.

The burn chain block that contains the candidates' commitment transaction
serves as the election block for the leader's block (i.e. _N_), and is used to
determine which block commitment "wins."

### Step 3: Sortition

In each election block, there is one election across all candidate leaders (across
all chain tips).  The next block is determined with the following algorithm:

```python
# inputs:
#   * BURNS -- a mapping from public keys to burn amounts and block hashes,
#              generated from the valid set of commit & burn transaction pairs.
# 
#   * PROOFS -- a mapping from public keys to their verified VRF proofs from
#               their election transactions.  The domains of BURNS and PROOFS
#               are identical.
#
#   * SEED -- the seed from the previous winning leader
#
# outputs:
#   * PUBKEY -- the winning leader public key
#   * BLOCK_HASH -- the winning block hash 
#   * NEW_SEED -- the new public seed

def make_distribution(BURNS):
   DISTRIBUTION = []
   BURN_OFFSET = 0
   for (PUBKEY, (BURN_AMOUNT, BLOCK_HASH)) in sorted(BURNS.items()):
      DISTRIBUTION.append((BURN_OFFSET, PUBKEY, BLOCK_HASH))
      BURN_OFFSET += BURN_AMOUNT
   return DISTRIBUTION

def select_block(SEED, BURNS, PROOFS, BURN_BLOCK_HEADER_HASH):
   if len(BURNS) == 0:
      return (None, None, hash(BURN_BLOCK_HEADER_HASH + SEED))

   DISTRIBUTION = make_distribution(BURNS)
   TOTAL_BURNS = sum(BURN_AMOUNT for (_, (BURN_AMOUNT, _)) in BURNS)
   SEED_NORM = num(SEED) / TOTAL_BURNS
   LAST_BURN_OFFSET = -1
   for (INDEX, (BURN_OFFSET, PUBKEY, BLOCK_HASH)) in enumerate(DISTRIBUTION):
      if LAST_BURN_OFFSET <= SEED_NORM and SEED_NORM < BURN_OFFSET:
         return (PUBKEY, BLOCK_HASH, hash(PROOFS[PUBKEY]))
      LAST_BURN_OFFSET = BURN_OFFSET
   return (DISTRIBUTION[-1].PUBKEY, DISTRIBUTION[-1].BLOCK_HASH, hash(PROOFS[DISTRIBUTION[-1].PUBKEY]))
```

Only one leader will win an election.  It is not guaranteed that the block the
leader produces is valid or builds off of the best Stacks fork.  However,
once a leader is elected, all peers will know enough information about the
leader's decisions that the block data can be submitted and relayed by any other
peer in the network.

Leaders can make their burn chain transactions and
construct their blocks however they want.  So long as the burn chain transactions
and block are broadcast in the right order, the leader has a chance of winning
the election.  This enables the implementation of many different leaders,
such as high-security leaders where all private keys are kept on air-gapped
computers and signed blocks and transactions are generated offline.

### On the use of a VRF

When generating the chain tip commitment transaction, a correct leader will need to obtain the
previous election's _seed_ to produce its proof output.  This seed, which is
an unbiased public random value known to all peers (i.e. the hash of the
previous leader's VRF proof), is inputted to each leader candidate's VRF using the private
key it committed to in its registration transaction.  The new seed for the next election is
generated from the winning leader's VRF output when run on the parent block's seed
(which itself is an unbiased random value).  The VRF proof attests that only the
leader's private key could have generated the output value, and that the value
was deterministically generated from the key.

The use of a VRF ensures that leader election happens in an unbiased way.
Since the input seed is an unbiased random value that is not known to
leaders before they commit to their public keys, the leaders cannot bias the outcome of the election 
by adaptively selecting proving keys.
Since the output value of the VRF is determined only from the previous seed and is 
pseudo-random, and since the leader already
committed to the key used to generate it, the leader cannot bias the new
seed value once they learn the current seed.

Because there is one election per burn chain block, there is one valid seed per
epoch (and it may be a seed from a non-canonical fork's chain tip).  However as
long as the winning leader produces a valid block, a new, unbiased seed will be
generated.

In the event that an election does not occur in an epoch, or the leader
does not produce a valid block, the next seed will be
generated from the hash of the current seed and the epoch's burn chain block header
hash.  The reason this is reasonably safe in practice is because the resulting
seed is still unpredictable and impractical (but not infeasible) to bias.  This is because the burn chain miners are
racing each other to find a hash collision using a random nonce, and miners who
want to attempt to bias the seed by continuing to search for nonces that both
bias the seed favorably and solve the burn chain block risk losing the mining race against
miners who do not.  For example, a burn chain miner would need to wait an
expected two epochs to produce two nonces and have a choice between two seeds.
At the same time, it is unlikely that there will be epochs
without a valid block being produced, because (1) attempting to produce a block
is costly and (2) users can easily form burning pools to advance the
state of the Stacks chain even if the "usual" leaders go offline.

# Operation as a leader

The Stacks blockchain uses a hybrid approach for generating block data:  it can
"batch" transactions and it can "stream" them.  Batched transactions are
anchored to the commitment transaction, meaning that the leader issues a _leading
commitment_ to these transactions.  The leader can only receive the block reward
if _all_ the transactions committed to in the commitment transaction
are propagated during its tenure.  The
downside of batching transactions, however, is that it significantly increases latency
for the user -- the user will not know that their committed transactions have been
accepted until the _next_ epoch begins.

In addition to sending batched transaction data, a Stacks leader can "stream" a
block over the course of its tenure by selecting transactions from the mempool
as they arrive and packaging them into _microblocks_.  These microblocks
contain small batches of transactions, which are organized into a hash chain to
encode the order in which they were processed.  If a leader produces
microblocks, then the new chain tip the next leader builds off of will be the
_last_ microblock the new leader has seen.

The advantage of the streaming approach is that a leader's transaction can be
included in a block _during_ the current epoch, reducing latency.
However, unlike the batch model, the streaming approach implements a _trailing commitment_ scheme.
When the next leader's tenure begins, it must select either one of the current leader's
microblocks as the chain tip (it can select any of them), or the current
leader's on-chain transaction batch.  In doing so, an epoch change triggers a
"micro-fork" where the last few microblocks of the current leader may be orphaned,
and the transactions they contain remain in the mempool.  The Stacks protocol
incentivizes leaders to build off of the last microblock they have seen (see
below).

The user chooses which commitment scheme a leader should apply for her
transactions.  A transaction can be tagged as "batch only," "stream only," or
"try both."  An informed user selects which scheme based on whether or not they
value low-latency more than the associated risks.

To commit to a chain tip, each correct leader candidate first selects the transactions they will
commit to include their blocks as a batch, constructs a Merkle tree from them, and
then commits the Merkle tree root of the batch
and their preferred chain tip (encoded as the hash of the last leader's
microblock header) within the commitment transaction in the election protocol.
Once the transactions are appended to the burn chain, the leaders execute
the third round of the election protocol, and the
sortition algorithm will be run to select which of the candidate leaders will be
able to append to the Stacks blockchain.  Once selected, the new leader broadcasts their
transaction batch and then proceeds to stream microblocks.

## Building off the latest block

Like existing blockchains, the leader can selet any prior block as its preferred
chain tip.  In the Stacks blockchain, this allows leaders to tolerate block loss by building
off of the latest-built ancestor block's parent.

To encourage leaders to propagate their batched transactions if they are selected, a
commitment to a block on the burn chain is only considered valid if the peer
network has (1) the transaction batch, and (2) the microblocks the leader sent
up to the next leader's chain tip commitment on the same fork.  A leader will not receive any compensation
from their block if any block data is missing -- they eventually must propagate the block data
in order for their rewards to materialize (even though this enables selfish
mining; see below).

The streaming approach requires some additional incentives to
encourage leaders to build off of the latest known chain tip (i.e. the latest
microblock sent by the last leader).  In particular, the streaming model enables
the following two safety risks that are not present in the batching approach:

* A leader who gets elected twice in a row can adaptively orphan its previous
  microblocks by building off of its first tenures' chain tip, thereby
double-spending transactions the user may believe are already included.

* A leader can be bribed during their tenure to omit transactions that are
  candidates for streaming.  The price of this bribe is much smaller than the
cost to bribe a leader to not send a block, since the leader only stands to lose
the transaction fees for the targeted transaction and all subsequently-mined
transactions instead of the entire block reward.  Similarly, a leader can
be bribed to mine off of an earlier microblock chain tip than the last one it has seen
for less than the cost of the block reward.

To help discourage both self-orphaning and "micro-bribes" to double-spend or
omit specific transactions or trigger longer-than-necessary micro-forks, leaders are
rewarded only 40% of their transaction fees in their block reward (including
those that were batched).  They receive
60% of the previous leader's transaction fees.  This result was shown in the
Bitcoin-NG paper to be necessary to ensure that honest behavior is the most
profitable behavior in the streaming model.

The presence of a batching approach is meant to raise the stakes for a briber.
Users who are worried that the next leader could orphan their transactions if
they were in a microblock would instead submit their transactions to be batched.
Then, if a leader selects them into its tenure's batch, the leader would
forfeit the entire block reward if even one of the batched transactions was
missing.  This significantly increases the bribe cost to leaders, at the penalty
of higher latency to users.  However, for users who need to send 
transactions under these circumstances, the wait would be worth it.

Users are encouraged to use the batching model for "high-value" transactions and
use the streaming model for "low-value" transactions.  In both cases, the use
of a high transaction fee makes their transactions more likely to be included in
the next batch or streamed first, which additionally raises the bribe price for
omitting transactions.

## Leader volume limits

A leader propagates blocks irrespective of the underlying burn chain's capacity.
This poses a DDoS vulnerability to the network:  a high-transaction-volume
leader may swamp the peer network with so many
transactions and microblocks that the rest of the nodes cannot keep up.  When the next
epoch begins and a new leader is chosen, it would likely orphan many of the high-volume
leader's microblocks simply because its view of the
chain tip is far behind the high-volume leader's view. This hurts the
network, because it increases the confirmation time of transactions
and may invalidate previously-confirmed transactions.

To mitigate this, the Stack chain places a limit on the volume of
data a leader can send during its epoch (this places a _de facto_ limit
on the number of transactions in a Stack block). This cap is enforced
by the consensus rules.  If a leader exceeds this cap, the block is invalid.

## Batch transaction latency

The fact that leaders execute a leading commmitment to batched transactions means that
it takes at least one epoch for a user to know if their transaction was
incorporated into the Stacks blockchain.  To get around this, leaders are
encouraged to to supply a public API endpoint that allows a user to query
whether or not their transaction is included in the burn (i.e. the leader's
service would supply a Merkle path to it).  A user can use a set of leader
services to deduce which block(s) included their transaction, and calculate the
probability that their transaction will be accepted in the next epoch.
Leaders can announce their API endpoints via the [Blockstack Naming
Service](https://docs.blockstack.org/core/naming/introduction.html).

The specification for this transaction confirmation API service is the subject
of a future SIP.  Users who need low-latency confirmations today and are willing
to risk micro-forks and intentional orphaning can submit their transactions for
streaming.

# Burning pools

Proof-of-burn mining is not only concerned with electing leaders, but also concerned with
enhancing chain quality.  For this reason, the Stacks chain not
only rewards leaders who build on the "best" fork, but also each peer who
supported the "best" fork by burning cryptocurrency in support of the winning leader.
The leader that commits to the winning chain tip and the peers who also burn for
that leader collectively share in the block's reward, proportional to how much
each one burned.

## Encouraging honest leaders

The reason for allowing users to support leader candidates at all is to help
maintain the chain's liveness in the presence of leaders who follow the
protocol correctly, but not honestly.  These include leaders who delay
the propagation of blocks and leaders who refuse to mine certain transactions.
By giving users a very low barrier to entry to becoming a leader, and by giving
other users a way to help a known-good leader candidate get selected, the Stacks blockchain
gives users a first-class stake in deciding which transactions to process
as well as incentivizes them to maintain chain liveness in the face of bad
leaders.  In other words, leaders stand to make more make money with
the consent of the users.

Users support their preferred leader by submitting a burn transaction that references 
its leader candidate's chain tip commitment.  These user-submitted burns count towards the
leader's total burn weight for the election, thereby increasing the chance
that they will be selected (i.e. users submit their transactions alongside the
leader's block commitment).  Users who burn for a leader that wins the election
will receive some Stacks tokens alongside the leader (but users whose leaders
are not elected receive no reward).  Users are rewarded the same way as
leaders -- they receive their tokens during the reward window (see below).

Allowing users to burn in support of leaders they prefer gives users and leaders
an incentive to cooperate.  Leaders can woo users to burn for them by committing
to honest behavior, and users can help prevent dishonest (but more profitable)
leaders from getting elected.  Moreover, leaders cannot defraud users who burn
in their support, since users are rewarded by the election protocol itself.

## Fair burning

Because all peers see the same sequence of burns in the Stacks blockchain, users
can easily set up distributed burn pools where each user receives a fair share
of the block rewards for all blocks the pool produces.  The selection of a
leader within the pool is arbitrary -- as long as _some_ user issues a key
transaction and a commitment transaction, the _other_ users in the pool can
throw their burns behind a chain tip.  Since users who burned for the winning
block are rewarded by the protocol, there is no need for a pool operator to
distribute rewards.  Since all users have global visibility into all outstanding
burns, there is no need for a pool operator to direct users to work on a
particular block -- users can see for themselves which block(s) are available by
inspecting the on-chain state.

Users only need to have a way to query what's going into a block when one of the pool
members issues a commitment transaction.  This can be done easily for batched
transactions -- the transaction sender can prove that their transaction is
included by submitting a Merkle path from the root to their transaction.  For
streamed transactions, leaders have a variety of options for promising users
that they will stream a transaction, but these techniques are beyond the scope of this SIP.

## Minimizing reward variance

Leaders compete to elect the next block by burning more cryptocurrency.
However, if they lose the election, the lose the cryptocurrency the burned.
This makes for a "high variance" pay-out proposition that puts leaders in a
position where they need to maintain a comfortable cryptocurrency buffer to
stay solvent.

To reduce the need for such a buffer, burning to support competing chain tips
enables leaders to hedge their bets by burning to support _all_ plausible
competing chain tips.  Leaders have the option of burning in support for a
_distribution_ of competing chain tips at a lower cost than committing to many
different chain tips as leaders.  This gives them the ability to receive some
reward no matter who wins, which reduces the expected amount of burn
cryptocurrency a leader needs to have on hand.  This also reduces the barrier to
entry for becoming a leader in the first place.

## Leader support mechanism

There are a couple important considerations for the mechanism by which peers
burn for their preferred chain tips.

* Users and runner-up leaders are rewarded strictly fewer tokens
for burning for a chain tip that does not get selected.  This is
  important because leaders and users are indistinguishable
on-chain.  Leaders should not be able to increase their expected reward by sock-puppeting,
and neither leaders nor users should get an out-sized reward for burning for
invalid blocks or blocks that will never be appended to the canonical fork.

* It must be cheaper for a leader to submit a single expensive commitment than it is
  to submit a cheap commitment and a lot of user burns.  This is
important because it should not be possible for a leader to profit more from
adaptively increasing their burns in response to other leader's burns.

The first property is enforced by the reward distribution rules (see below),
whereby a burn only receives a reward if its block successfully extended the
"canonical" fork.  The second property is given "for free" because the underlying burn chain
assesses each participant a transaction fee.  Users and leaders incur an ever-increasing
cost of trying to adaptively out-burn other leaders by submitting more and more
transactions.

Peers who want to support a leader candidate must send their burn transactions _in the
same burn chain block_ as the commitment transaction.  This limits the degree to
which peers can adaptively out-bid each other to include their
commitments.  However, this constraint creates an undesirable
negative feedback loop for supporting
leaders -- if there is _too much_ interest in a leader, then peers may accidentally 
kick their preferred commitment out of the target burn block (or kick all
burn commitments out).  Because a commitment transaction may only be accepted if
it is mined in one block, accidentally bumping it out of the leader's targeted
block will waste everyone's cryptocurrency and dissuade
users from supporting a leader in the future.

While there is no way to prevent this "burn collapse"
scenario outright, the Stacks protocol helps reduce the likelihood of it
happening through two mechanisms.  First, Stacks does not implement an all-or-nothing reward system --
peers can hedge their bets by committing to 
a distribution of chain tips, so they only need to send one burn transaction per chain tip candidate
in order to receive _some_ reward instead of
feeling compelled to one-up each other.  Second, the fork selection rules
(described below) ensure that burns can only help leader so much -- they stop helping after a certain
threshold, and too many burns over a short time hurts the leader's chances in future
elections.

# Reward distribution

New Stacks tokens come into existence on a fork in an epoch where a leader is
selected, and are granted to the leader if the leader produces a valid block.
However, the Stacks blockchain pools all tokens created and all transaction fees received and
does not distribute them until a large number of epochs (a _lockup period_) has
passed.  The tokens cannot be spent until the period passes.

The idealized payout distribution in a proof-of-burn blockchain is a
distribution where each burner receives tokens proportional to the fraction of burns they
contributed for the block.  In addition to implementing a lockup period, the
Stacks protocol "smooths out" the token rewards over a _reward window_
to help realize a close approximation of this payout distribution.

## Sharing the rewards among winners

Winning blocks and the leaders and users who burned for them are not rewarded
in a winner-take-all fashion.  Instead, after the
rewards are delayed for a lock-up period, the are distributed to all winning
burns over a _reward window_.  The block rewards
are allotted to each burning participant incrementally as the window passes,
based on the ratio between how
much it burned over the window versus how much everyone
burned over the window.  This has the (desired) effect of "smoothing out" the
rewards so as to minimize the difference between the idealized earnings and actual earnings
across all winning participants.

The "smoothing" mechanism is necessary to keep peers incentivized to mine a
maximal number of transactions when either the Stacks block rewards
or the burn chain block rewards are dominated by transaction fees instead
of coinbases.  On the Stacks chain, the fact that the peers all share the transaction
fees proportionally means that they are incentivized to mine a maximal number of
transactions during their epochs, no matter how big they are relative to the
coinbase.  This avoids some of the [chain instability](http://randomwalker.info/publications/mining_CCS.pdf)
problems that arise in winner-takes-all payout schemes, whereby
miners implement a "petty compliant" strategy where a minimal amount of
transactions are mined per block once transaction fees exceed coinbase rewards.

On the burn chain, the presence of commitment transactions means that each burn
chain miner _who also participates as a Stacks leader_ stands to receive both a
burn chain block reward and a Stacks block reward.  This means that at a minimum, a
burn chain miner is incentivized to (1) participate as a Stacks leader in order
to increase their total revenue (even in the presence of dysfunctional miner
strategies), and (2) a burn chain miner is incentivized to at
least include their own burns in every burn block the mine, as well as build off of
the previous burn chain tip that also includes their preferred Stacks chain tip.
In other words, if the burn chain's transaction fees exceed its coinbase, then
the Stacks leader election process is expected to "degrade" to a situation where
only burn chain miners are Stacks leaders (i.e. they only mine their own Stacks
burn chain transactions), but they all mine Stacks burn transactions and
keep the burn chain live enough for the Stacks chain to make progress (i.e.
economically rational miners don't reorg the burn chain because they maximize
their Stacks reward by (1) working on top of previous valid Stacks chain tips
and (2) ensuring that previously-accepted Stacks chain tips stay accepted).

# Recovery from data loss

Stacks block data can get lost after a leader commits to it.  However, the burn
chain will record the chain tip, the batched transactions' hash, and the leader's public
key.  This means that all existing forks will be
known to the Stacks peers that share the same view of the burn chain (including
forks made of invalid blocks, and forks that include blocks whose data was lost
forever).

What this means is that regardless of how the leader operates, its
chain tip commitment strategy needs a way to orphan a fork of any length.  In
correct operation, the network recovers from data loss by building an
alternative fork that will eventually become the "best" fork,
thereby recovering from data loss and ensuring that the system continues
to make progress.  Even in the absence of malice, the need for this family of strategies
follows directly from a single-leader model, where a peer can crash before
producing a block or fail to propagate a block during its tenure.

However, there is a downside to this approach: it enables **selfish mining.**  A
minority coalition of burns can statistically gain more Stacks tokens than they are due from
their burn amounts by attempting to build a hidden fork of blocks, and releasing it
once the honest majority comes within one block height difference of the hidden
fork.  This orphans the majority fork, causing them to lose their Stacks tokens
and re-build on top of the minority fork.

## Seflish mining mitigation strategies

Fortunately, all peers in the Stacks blockchain have global knowledge of state,
 time, and burns.  Intuitively, this gives the Stacks blockchain some novel tools
for dealing with selfish leaders:

* Since all nodes know about all blocks that have been committed, a selfish leader coalition
  cannot hide its attack forks. The honest leader coalition can see
the attack coming, and evidence of the attack will be preserved in the burn
chain for subsequent analysis.  This property allows honest leaders
to prepare for and mitigate a pending attack by burning more
cryptocurrency, thereby reducing the fraction of burn power the selfish leaders wield
below the point where selfish mining is profitable (subject to network
conditions).

* Since all nodes have global knowledge of the passage of time, honest leaders
  can agree on a total ordering of all chain tip commits and burns.  In certain kinds of
selfish mining attacks, this gives honest leaders the ability to identify and reject an attack fork 
with over 50% confidence.  In particular, honest leaders who have been online long
enough to measure the expected block propagation time would _not_ build on top of
a chain tip whose last _A > 1_ blocks arrived late, even if that chain tip
represents the "best" fork, since this would be the expected behavior of a selfish miner.

* Since all nodes know about all burn transactions, the long tail of small burners
(i.e. users who support leaders) can collectively throw their burns behind
known-honest leaders' burns.  This increases the chance that honest leaders will
be elected, thereby increasing the fraction of honest burn power and making it
harder for a selfish leader to get elected.

* The Stacks chain reward system spreads out rewards for creating
blocks and mining transactions across a large interval.  This "smooths over"
short-lived selfish mining attacks -- while selfish leaders still receive more
than their fair share of the rewards, the lower variance imposed by the
reward window makes this discrepancy smaller.

* All Stacks nodes relay all blocks that correspond to on-chain commitments,
even if they suspect that they came from the attacker.  If an honest leader finds two chain tips of equal
length, it selects at random which chain tip to build off of.  This ensures that
the fraction of honest burns that build on top of the attack fork versus the honest fork
is statistically capped at 50% when they are the same length.

None of these points _prevent_ selfish mining, but they give honest users and
honest leaders the tools to make selfish mining more difficult to pull off than in
PoW chains.  Depending on user activity, they also make economically-motivated
leaders less likely to participate in a selfish miner cartel -- doing so always produces evidence,
which honest leaders and users can act on to reduce or eliminate 
their expected rewards.

Nevertheless, these arguments are only intuitions at this time.  A more
rigorous analysis is needed to see exactly how these points affect the
profitibility of selfish mining.  Because the Stacks blockchain represents a
clean slate blockchain design, we have an opportunity to consider the past
several years of research into attacks and defenses against block-hiding
attacks.  This section will be updated as our understanding evolves.

# Fork Selection

Fork selection in the Stacks blockchain requires a metric to determine which
chain, between two candidates, is the "best" chain.  Using proof-of-burn as the
security method for the blockchain implies a direct metric:  the total sum of
burns in the election blocks for a candidate chain.  In particular, **the Stacks
blockchain measures a fork's quality by the total amount of burns which _confirms_ block _N_** (as
opposed to the amount of burn required for the _election_ of block _N_).

This fork choice rule means that the best fork is the _longest valid_ fork.
This fork has the most valid blocks available of all forks, and over time will have the
highest cumulative proof-of-burn of all forks.  This is
because a fork that has the most consecutive blocks will, with high probability,
be produced by a succession of correct leaders who at the time of their election
are selected from the set of candidates backed by the majority of the burn rate.

Using chain length as the fork choice rule makes it time-consuming for alternative forks to
overtake the "canonical" fork, no matter how much burn capacity they have.
In order to carry out a deep fork, the majority coalition of burns needs to spend
at least as many epochs working on the new fork as they did on the old fork.
We consider this acceptable because it also has the effect of keeping the chain
history relatively stable, and makes it so every participant can observe (and
prepare for) any upcoming forks that would overtake the canonical history.  However, a minority
coalition of dishonest leaders can cause short-lived reorgs by continuously
building forks (i.e. in order to selfishly mine), driving up the confirmation
time for transactions in the honest fork.

An alternative fork-selection rule was considered whereby the chain with the
most total burns would have been the "best" chain, no matter how long it was.
This idea was ultimately
rejected because it would mean that a single rich leader could invalidate a
large number of blocks with a single massive burn.  This is not only an unacceptable risk
to proof-of-burn blockchains that are just getting off the ground, but also 
does not reward liveness as well -- i.e. users want the chain history 
to be relatively stable and to usually make forward progress.

## Burn Window

The rate at which the burn tokens are destroyed for proof-of-burn mining
will fluctuate over time.  This affects the ease at which a
"deep fork" can be executed:

* If the burn rate increases too quickly, then a few rich leaders can quickly dominate the
  sortition process (and rewards) and effectively take over the chain before other
participants have had a chance to react.  A set of rich leaders could burn a
large amount of burn tokens to produce an alternative fork in only a few rounds (on the order of the
depth of the fork) and maintain it as long as they have at least 51% of the burn
capacity.   Also, if there are too many commitment
transactions or burn transactions in the burn chain mempool, a "burn collapse"
event can result whereby a legitimate commitment transaction is prevented from
being mined and the Stacks blockchain stalls for the epoch.

* If the burn rate decreases too quickly, then it makes it easy for opportunistic attackers to
  cheaply produce an alternative fork before honest participants can react.  In particular, 
an opportunistic fork coalition can create a deep fork in only a few rounds in a
similar manner described above.

In order to increase the time it takes to execute a deep fork under these
circumstances (i.e. to give the network's participants a chance to react),
the Stacks blockchain implements a "burn window" whereby it
decides how much cryptocurrency all participants must destroy in order for 
*any* leaders to be selected (regardless of which chain tip).
This measurement includes **all** block commitments and **all** user burns --
even well-formed burns for otherwise invalid or missing blocks.

The burn window is measured over a variable-length sequence of epochs and has a "burn quota"
that must be met before a leader can be elected.  The burn quota determines how
much cryptocurrency must be burned in all epochs in the window in order for the
next sortition to occur.  The burn quota is controlled via a negative feedback loop, whereby
the burn quota is additively increased with excessive burns and
multiplicatively decreased in the absence of burns.  The Stacks
blockchain tracks an average burns/window value to determine which action to
take on the burn quota on the arrival of the next burn-chain block.

### Adjusting the window

As more cryptocurrency is burned, the burn quota of
the burn window increases additively.  If adding the next block to the burn
window sufficiently increments the window's average burn/block ratio from the
last time the burn quota was adjusted,
then the burn quota is incremented by a protocol-defined constant.

As less cryptocurrency is burned, adding the next
block to the burn window would decrease its window's burns/block ratio.  If the
burns/block ratio falls beneath a fixed
fraction of its maximum value since its last reduction (e.g. it falls beneath 80% of
the last maximum burn quota), then
the burn window "grows" to include the next burn chain block.  No leader will be elected
for its corresponding epoch.  The window contiunes to grow in this manner, up until the sum of its burns
in the window exceeds the burn quota.  Once it is met, a new leader will be selected, and the window will "snap
back" to its original size.  The burn quota will be decreased multiplicatively
once this happens (e.g. reduced by 25%).

If the cryptocurrency burn rate remains stable -- that is, the average
burn/block ratio stays between the value at which the burn quota decreases
and the value at which the burn quota increases, then no adjustment takes place.

## Effects on sortition

In normal operation, tracking a burn window enables a set of leader candidates to
burn cryptocurrency units at a rate that is about the market rate for Stacks
tokens.  The feedback loop that governs the window's burn quota
creates a steady-state behavior where there is about one Stacks block produced per
epoch, even in the face of wild adjustments
in the market prices of Stacks and the underlying burn cryptocurrency, and in
the face of the rise and fall in popularity of non-canonical forks.

In the absence of a leader election (if the burn quota is not met), the block seed
that will be used for the next leader election will be calculated
per usual -- at each subsequent no-leader epoch, the new seed will be calculated as the hash
of the current seed and the burn block header's hash.  As argued earlier, this
ensures that the next election, when it occurs, will be unbiased.

Under adversarial conditions where multiple forks are competing, the burn window
gives the sortition algorithm a way to prevent a rich coalition from quickly taking
control of the sortition algorithm.  This occurs in two ways.  First, if an _individual chain tip_ accumulates a
protocol-defined "high" fraction of the average burn/block ratio, then its
associated weight in the sortition algorithm is capped.  Second, if the _total burn_ exceeds
a (different) protocol-defined "high" fraction of the average burn/block ratio,
then the sortition algorithm will _only_ select blocks that build on the
canonical fork's chain tip.  What this does is ensure that a rich coalition
working on an alternative fork cannot immediately gain a greater-than-50%
chance of getting selected -- it must
first additively increase the average burn/block ratio to the point where it can
both out-burn the remainder of the leaders and work on the alternative fork.
This gives the rest of the network time to react to it -- the burn quota can be
made to increase slowly, so a rich coalition would need to work on the
alternative fork for days or even weeks.

At the same time, the burn window gives the sortition algorithm the ability to
filter out burns that are too small -- i.e. burns that are smaller than a
"low" fraction of the average burn/block ratio.  Filtering out low burns
serves two purposes:  ensuring that "spam attacks" are not profitable, and
discouraging the conditions that lead to a "burn collapse."  Regarding spam
attacks, filtering out low burns ensures that leaders of forks with very little 
burn support have zero chance of being selected by the sortition algorithm.  This prevents
a liveness failure scenario whereby a large number of low-burning peers DDoS the
Stacks chain by committing to many different chain tips forks and collectively
out-burning the honest coalition of leaders (at very small individual cost).
This also effectively requires any fork that has a chance of becoming the
"canonical" fork to have a large fraction of the burns behind them for a long period of time.

Regarding burn collapse, ignoring burns that are too small encourages users to
combine small burns into a single large burn when possible.  This in turn helps
avoid the scenario whereby a legitimate chain commitment transaction gets bumped
from its target block due to too much support.  When there is wide-spread
interest in a particular chain tip from users, filtering small burns provides an
economic incentive for users to make judicious use of the limited burn block
space.

# Implementation

The Stacks blockchain leader election protocol will be written in Rust.

## Leader election protocol burn-chain wire formats

```
Key transaction wire format
0      2  3              19                       51                          80
|------|--|---------------|-----------------------|---------------------------|
 magic  op consensus hash   proving public key                   memo

Commitment transaction wire format
0      2  3              35                 67     71     73    77   79       80
|------|--|---------------|-----------------|------|------|-----|-----|-------|
 magic  op   block hash       new seed       parent parent key   key    memo
                                             delta  txoff  delta txoff 

User support transaction wire format
0      2  3              19                       51                 71       80
|------|--|---------------|-----------------------|------------------|--------|
 magic  op consensus hash    proving public key       block hash 160    memo

Field name        |   Field contents
------------------|-------------------------------------------------------------------------------------------
magic             |  network ID (e.g. "id")
op                |  one-byte opcode that identifies the transaction type
proving public key|  EdDSA public key (32 bytes)
block hash        |  SHA256(SHA256(block header))
block hash 160    |  RIPEMD160(block hash)
consensus hash    |  first 16 bytes of RIPEMD160(merkle root of prior consensus hashes)
parent delta      |  number of blocks back from this block in which the parent block header hash can be found
parent txoff      |  offset in the block that contains the parent block header hash
key delta         |  number of blocks back from this block in which the proving public key can be found
key txoff         |  offset in the block that contains the proving public key
new seed          |  SHA256(SHA256(parent seed ++ VRF proof))
memo              |  arbitrary data
```

# Appendix

## Definitions

**Burn-mining** is the act of destroying cryptocurrency from one blockchain in
order to mine a block on another blockchain.  Destroying the cryptocurrency can
be done by rendering it unspendable.

**Burn chain**: the blockchain whose cryptocurrency is destroyed in burn-mining.

**Burn transaction**: a transaction on the burn chain that a Stacks miner issues
in order to become a candidate for producing a future block.

**Chain tip**: the location in the blockchain where a new block can be appended.
Every valid block is a valid chain tip, but only one chain tip will correspond
to the canonical transaction history in the blockchain.  Miners are encouraged
to append to the canonical transaction history's chain tip when possible.

**Cryptographic sortition** is the act of selecting the next leader to
produce a block on a blockchain in an unbiased way.  The Stacks blockchain uses
a _verifiable random function_ to carry this out.

**Election block**: the block in the burn chain at which point a leader is
chosen.  Each Stacks block corresponds to exactly one election block on the burn
chain.

**Epoch**: a discrete configuration of the leader and leader candidate state in
the Stacks blockchain.  A new epoch begins when a leader is chosen, or the
leader's tenure expires (these are often, but not always, the same event).

**Fork**: one of a set of divergent transaction histories, one of which is
considered by the blockchain network to be the canonical history
with the "best" chain tip.

**Fork choice rule**: the programmatic rules for deciding how to rank forks to select
the canonical transaction history.  All correct peers that process the same transactions
with the same fork choice rule will agree on the same fork ranks.

**Leader**: the principal selected to produce the next block in the Stacks
blockchain.  The principal is called the block's leader.

**Reorg** (full: _reorganization_): the act of a blockchain network switching
from one fork to another fork as its collective choice of the canonical
transaction history.  From the perspective of an external observer,
such as a wallet, the blockchain appears to have reorganized its transactions.
