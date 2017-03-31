# Extension Blocks

```
Layer: Consensus (soft-fork)
Title: Extension Blocks
Author: Christopher Jeffrey <chjj@purse.io>
        Joseph Poon <joseph@lightning.network>
        Fedor Indutny <fedor@indutny.com>
Status: Draft
Created: 2017-03-17
License: Public Domain
```

## Abstract

This specification defines a method of increasing bitcoin transaction
throughput without altering any existing consensus rules.

## Motivation

Bitcoin retargetting ensures that the time in between mined blocks will be
roughly 10 minutes. It is not possible to change this rule. There has been
great debate regarding other ways of increasing transaction throughput, with no
proposed consensus-layer solutions that have proven themselves to be
particularly safe.

## Specification

Extension blocks devise a second layer on top of canonical bitcoin blocks in
which a miner will commit to the merkle root of an additional block of
transactions.

Extension blocks leverage several features of BIP141, BIP143, and BIP144 for
transaction opt-in, serialization, verification, and network services, and as
such, extension block activation entails BIP141 activation.

This specification should be considered an extension and modification to these
BIPs. Extension blocks are _not_ compatible with BIP141 in its current form,
and will require a few minor additional rules.

Extension blocks maintain their own UTXO set in order to avoid interference
with the existing UTXO set of non-upgraded nodes. This is a tricky endeavor,
requiring a `resolution` transaction to be present at the end of every
canonical block.

This specification prescribes a way of fooling non-upgraded nodes into
believing the existing UTXO set is still behaving as they would expect.
Unfortunately, this requires a bit of extra book-keeping on the upgraded node's
side.

### Commitment

An upgraded miner willing to include entering outputs and/or an extension block
is to include an extra coinbase output of 0 value. The output script exists as
such:

```
OP_RETURN 0x24 0xaa21a9ef[32-byte-merkle-root]
```

The commitment serialization and discovery rules follows the same rules defined
in BIP141.

The merkle root is to be calculated as a merkle tree with all extension block
txids and wtxids as the leaves. Regular txids, although not necessary for
security purposes, are included for the possibility of regular TXID merkle
proofs.

Canonical blocks containing entering outputs MUST contain an extension block
commitment (all zeroes if nothing is present in the extension block).

### Extension block opt-in

Outputs can signal to enter the extension block by using witness program
scripts (specified in BIP141). Outputs signal to exit the extension block if
the contained script is either a minimally encoded P2PKH or P2SH script.

Output script code aside from witness programs, p2pkh or p2sh is considered
invalid in extension blocks.

### Resolution

Resolution transactions are a book-keeping mechanism which exist to maintain
the total value of the extension block UTXO set. They exist as a consecutively
redeemed OP_TRUE output. They handle both entrance into and exits from the
extension block UTXO set.

Every block containing an extension block commitment MUST contain a final
`resolution` transaction. This resolution transaction spends all outputs that
intend to enter the extension block. The resolution transaction MUST appear as
the last transaction in the block (in order to properly sweep all of the newly
created outputs). The funds are to be sent to a single anyone-can-spend
(OP_TRUE) output. This output is forbidden by consensus rules to be spent by
any transaction aside from another `resolution` transaction.

The resolution transaction MUST contain additional outputs for outputs that
intend to exit the extension block.

Coinbase outputs MUST NOT contain witness programs, as they cannot be sweeped
by the resolution transaction due to previously existing consensus rules.

The first input of a resolution transaction MUST reference the first output of
the previous resolution transaction.

In order to bootstrap the activation of extension blocks, a "genesis"
resolution transaction MUST be mined in the first block to include an extension
block commitment along with any entering outputs (the `activation` block). This
is the only resolution transaction in existence that does not require a
reference to a previous resolution transaction.

Fees are to propagate up from the extension block into the resolution
transaction. In other words, the resolution transaction's fee should should be
equal to the amount of total fees collected in the extension block.

#### Resolution Rules

The resolution transaction's first output MUST have a value equal to:

`(previous-resolution-value + entering-value - exiting-value) - ext-block-fees`

The following output scripts and values must replicate the extension block's
exiting outputs _exactly_ (in the same order they appear in the ext. block).

The resolution transaction's version MUST be set to the uint32 max
(`2^32 - 1`). This version is forbidden by consensus rules to be used with any
other transaction on the canonical chain or extension chain. This is required
for easy non-contextual identification of a resolution transaction.

### Entering an extension block

Any witness program output is considered an opt-in to fund a keyhash or
scripthash within the extension block's UTXO set.

Example block:

```
Transaction #1 (coinbase):

Output #0:
  - Script: P2PKH
  - Value: 12.5
Output #1:
  - Script: OP_RETURN 0xaa21a9ef[merkle-root]
  - Value: 0
```

```
Transaction #2 (extension block funding transaction):

Output #0:
  - Script: P2WPKH (will enter the extension utxo set)
  - Value: 5.0

Output #1:
  - Script: P2PKH (stays in the canonical utxo set)
  - Value: 2.5
```

The resolution transaction shall redeem any witness program outputs:

```
Transaction #3 (resolution transaction):

Input #0:
  - Outpoint:
    - Hash: previous-resolution-txid
    - Index: 0

Input #1:
  - Outpoint:
    - Hash: Transaction #2 TXID
    - Index: 0

Output #0:
  - Script: OP_TRUE
  - Value: 5.0
```

In the case that a spender wants to redeem tx2/output0, the corresponding
extension block may exist as such:

```
Transaction #4 (redeemer of transaction #2):

Input #0:
  - Outpoint:
    - Hash: Transaction #2 TXID
    - Index: 0

Output #0:
  - Script: P2WPKH (this output remains in the ext. block)
  - Value: 5.0
```

### Exiting an extension block

In order to ensure a 1-to-1 value between the existing blockchain and the
extension block, an exit must be provided for outputs that exist in the
extension block.

In a later transaction, the spender of Transaction #2's output may want an
output to exit with some of their money.

```
Transaction #5 (coinbase):

Output #0:
  - Script: P2PKH
  - Value: 12.5

Output #1:
  - Script: OP_RETURN 0xaa21a9ef[merkle-root]
  - Value: 0
```

```
Transaction #6 (resolution transaction):

Input #0:
  - Outpoint:
    - Hash: previous-resolution-txid
    - Index: 0

Output #0:
  - Script: OP_TRUE
  - Value: 2.5

Output #1:
  - Script: P2PKH (from the exited output below)
  - Value: 2.5
```

Extension block:

```
Transaction #7:

Input #0:
  - Outpoint:
    - Hash: Transaction #4 TXID
    - Index: 0

Output #0:
  - Script: P2WPKH (this output will remain in the ext. block)
  - Value: 2.5

Output #1:
  - Script: P2PKH (note that this causes an exit!)
  - Value: 2.5
```

### Fees

Fees collected from inside the extension block propagate up to the
corresponding resolution transaction. The resolution transaction's fee MUST be
equal to the cumulative amount of fees collected inside the extension block.

On the policy layer, transaction fees may be calculated by transaction cost as
well as additional size/legacy-sigops added to the canonical block due to
entering or exiting outputs.

In the previous example, the spender of Transaction #2's output could have also
added a fee.

```
Transaction #5 (coinbase):

Output #0:
  - Script: P2PKH
  - Value: 12.501 (reward + fee)

Output #1:
  - Script: OP_RETURN 0xaa21a9ef[merkle-root]
  - Value: 0
```

```
Transaction #6 (resolution transaction):

Input #0:
  - Outpoint:
    - Hash: previous-resolution-txid
    - Index: 0

Output #0:
  - Script: OP_TRUE
  - Value: 2.499 (fee is subtracted)

Output #1:
  - Script: P2PKH (from the exited output below)
  - Value: 2.5
```

Extension block:

```
Transaction #7:

Input #0:
  - Outpoint:
    - Hash: Transaction #4 TXID
    - Index: 0

Output #0:
  - Script: P2WPKH (this output will remain in the ext. block)
  - Value: 2.499 (fee is subtracted, this propagates up)

Output #1:
  - Script: P2PKH (note that this causes an exit!)
  - Value: 2.5
```

### BIP141 Rule Changes

- Aside from the resolution transaction, witness program outputs are only
  redeemable inside of an extension block.
- Witness transactions may _only_ contain witness program inputs.
- BIP141's nested P2SH feature is no longer available, and no longer a
  consensus rule.
- The concepts of `block weight` and `transaction weight` have been removed.
  Size is once again the metric to be used for calculating fees/dos limits/etc.
- The concept of `sigops cost` remains present for future soft-forkable and
  upgradeable DoS limits.
- Any extended transaction MUST have a version with the 31st bit set to `1`.
  Transaction version 2 on the extension chain would exist as `(1 << 30) | 2`.
  This bit MUST NOT be used in canonical chain transactions. This is to provide
  an easy non-contextual way of identifying extension chain transactions.

### DoS Limits

DoS limits shall be enforced by extension block size as well as the newly
defined metrics of inputs and outputs cost. Be aware that exiting outputs
inside the extension block affect DoS limits in the canonical block, as they
add size and legacy sigops.

```
MAX_BLOCK_SIZE: 1000000 (unchanged)
MAX_BLOCK_SIGOPS: 20000 (unchanged)
MAX_EXTENSION_SIZE: 5000000
MAX_EXTENSION_COST: TBD
```

The maximum extension size is intentionally high. The average case block is
truly limited by inputs (sigops) and outputs cost.

Future size and computational scalability can be soft-forked in with the
addition of new witness programs. On non-upgraded nodes, unknown witness
programs count as 0 sigops/outputs cost. Lesser cost can be implemented for
newer witness programs in order to allow future soft-forked dos limit changes.

##### Extented Transaction Cost

Extension blocks leverage BIP141's upgradeable script behavior to also allow
for upgradeable DoS limits.

###### Calculating Inputs Cost

Witness key hash v0 shall be worth 8 points.

Witness script hash v0 shall be worth the number of accurately counted sigops
in the redeem script, multiplied by 8. To reduce the chance of having redeem
scripts which simply allow for garbage data in the witness vector, every 73
bytes in the serialized witness vector is worth an accurate sigop as well.

Unknown witness programs shall be worth 1 point with an additional point for
every 73 bytes in the serialized witness vector.

This leaves room for 7 future soft-fork upgrades to relax DoS limits.

###### Calculating Outputs Cost

Currently defined witness programs (v0) are each worth 8 points. Unknown
witness program outputs are worth 1 point. Any exiting output is always worth
8 points.

This leaves room for 7 future soft-fork upgrades to relax DoS limits.

#### Dust Threshold

A consensus dust threshold is now enforced within the extension block.

Outputs containing less than 500 satoshis of value are _invalid_ within an
extension block. This _includes_ entering outputs as well, but not exiting
outputs.

### Consensus rules for extra Lightning security

If the second highest transaction version bit (30th bit) is seto to `1`, then
extra blockspace and sigops cost is allocated towards two transactions. The
space limit being preallocated is two transactions of 300 bytes (for a total of
600 bytes).

The first allocation can only be consumed by a transaction which directly spends
from the first output of this transaction.

The second allocation can only consume the first output of any transaction
within the past 2016 blocks which have the 30th bit set.

If the allocation is not used by other transactions, then the transaction
consumes that extra space, reducing the blocksize by up to 600 bytes in
available space.

This is a consensus rule within the extension block and does not apply to the
main-chain block.

The purpose is to ensure that without miner coordination, the costs will be
unusually high for systemic attacks, since blockspace is preallocated for miners
to include penalty Lightning Network transactions, and this is enforced since
both parties must agree to the version bit in Commitment Transaction in
Lightning. This is an opt-in feature in Lightning, and trades higher transaction
fees for greater availability for penalties. The assumption is in the majority
case of incorrect broadcast, that the penalty will go into the same block via
the second allocation, and to give room for other transactions in the first
allocation.

This prevents specific types of systemic attacks as defined in the Lightning
Network whitepaper risks.

### Backward Compatibility (consensus)

TODO

### Migration and adoption concerns

Most of the bitcoin ecosystem is currently well-equipped to handle an upgrade
to BIP141.

For wallets currently supporting BIP141, the migration should be trivial.

For fullnode-based services, APIs may be altered to transparently serve
extension block transactions to clients as if they appeared in the canonical
block. This, of course, would not include any miner API.

#### Wallet Migration

Wallets currently supporting BIP141 must be modified in a few key ways:

1. Transactions must pick a chain to spend from (either the canonical block or
   the extension block, but not both). In other words, transactions must have all
   witness program inputs, or all non-witness program inputs. For wallets that
   support both chains, the coin selector can automatically pick which chain to
   use if the user does not specify.

2. Wallets supporting ext-blocks/BIP141 must ignore inputs on resolution
   transactions if seen. This should be a simple check of transaction version
   number, and similar to how wallets already ignore a coinbase's input. This
   is necessary to prevent wallets from mistakenly seeing a double spend.

3. Wallets supporting both canonical block and extension block funds must
   ignore exiting outputs within the extension block. This is necessary to
   prevent wallets from mistakenly indexing the same output twice.

### Mempool Concerns

Changes to mempool implementations are surprisingly minimal. Although details
may differ across implementations, a conforming mempool must disallow
cross-chain spends (mixed inputs on both chains), as well as track exiting
output's outpoints. These outpoints may not be spent inside the mempool (they
must be redeemed from the next resolution txid in reality).

### Mining Concerns

#### Additional size and sigops calculation

Nodes creating block templates internally will have to take into account
entering and exiting outputs when reserving size for the canonical block. A
transaction with entering outputs or exiting outputs adds size to the canonical
block.

Entering outputs add size in the form of inputs in the resolution transaction.

Exiting outputs add both size and legacy sigops in the form of duplicated
outputs on the resolution transaction.

Transaction sorting and selection algorithms must take care to account for
this.

#### Extensions to `getblocktemplate` (BIP22)

In addition to the `transactions` array, conforming implementations MUST
include an `extension` array. The `extension` array contains transactions as
defined in BIP22 as well as the BIP145 extensions to `getblocktemplate`.

Conforming implementations MUST include the resolution transaction as part of
the `transactions` array. The resolution transaction must have an extra
property called `resolution`, which is a boolean value set to `true`. Including
the resolution TX in the `transactions` array directly is done for backward
compatibility with existing mining clients.

`default_witness_commitment` has been renamed to `default_extension_commitment`
and includes the extension block commitment script.

An extra `mutable` field is defined for `"resolution"`. If `transactions` is
present in the mutable array, `resolution` allows extension block mutation,
provided the client is able to properly update the resolution transaction.

Along with the `resolution` mutable field, a `resolution` `capabilities` field
is also defined in order for the client to state that it is capable of updating
the resolution transaction.

For nodes dealing with out of date mining clients, storing the extension block
locally in a map of `commitment-hash->ext. block` may be required (along with
some reserialization during a `submitblock` call).

### Activation

- Version bit: 2
- Deployment name: `extblk` (appears as `!extblk` in GBT).
- Start time: 1491004800 (April 1st, 2017 UTC)
- Timeout: 1522540800 (April 1st, 2018 UTC)

### Deactivation

The extension block MUST deactivate 210240 blocks (around 4 years) after the
activation of the extension block.

After deactivation, exits from the extension block are still possible, but
entrance into and transfer of funds within the extension block is no longer
possible.

By this point, a future extension block ruleset will likely have been
developed, which is superior in terms of feature-set and scalability (see also:
Rootstock and/or Mimblewimble).

Redemption from the old extension block to the new extension block can be
migrated by way of merkle proofs. Funds may be imported to the new extension
block by hard-coding a UTXO merkle root into the implementation as a consensus
rule, and verifying imported funds against a merkle path.

Nodes only need to have a copy of the current 32-byte merkle root of the
deactivated extension block.

This allows for fullnodes to not need to store a copy of the UTXO set on disk,
nor keep a cache of the UTXO in memory/disk, with a tradeoff of larger on-chain
transactions when redeeming in the future. An alternative would be for all
clients to maintain a record of the UTXO set and keep the full bitfield in
memory.

Since the TXO set is static (with only whether it is unspent or spent changing)
as it is deactivated from new outputs, this is simpler than designs for changing
UTXOs:
https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2015-October/011638.html

In order to make this soft-forkable, the fund pool amount locked on the main
chain allocated to the new extension blockchain is combined with the deactivated
one. This allows for a direct transfer without affecting main-chain consensus
rules (without a hard fork). The actual accounting of the allocation between the
deactivated extension blockchain and the new one is maintained by the individual
nodes, so it can follow any future soft-fork rule with transfers to the new
extension blockchain.

#### Motivation

While deactivation may be seen as a drawback, the primary benefit of this
specification is _not_ an extension block with a BIP141-ruleset. Instead, it is
that the ecosystem will have laid the groundwork for handling extension blocks.
The future of the bitcoin protocol may include several different extension
blocks with many different rulesets.

## Testnet (extnet)

A live testnet is currently running with a full implementation of extension
blocks as of [DATE].

Seed: TBA

## Reference Implementation

- https://github.com/bcoin-org/bcoin-extension-blocks

---

# Extension Blocks (peer services)

```
Layer: Peer Services
Title: Extension Blocks (peer services)
Author: Christopher Jeffrey <chjj@purse.io>
Status: Draft
Created: 2017-03-17
License: Public Domain
```

## Abstract

TODO

## Specification

Note: Separate network messages for ext-tx vs tx? ext-block vs block?

### Peer Services

Extension blocks defines a new service bit (service bit 5) to signal whether
extension blocks is supported by the remote node.

### Block and transaction relay

This specification defines new INV types for GETDATA requests. Similar to the
INV types defined in BIP144, an `extended bit` is to be set on both `tx` and
`block` inv types. The extended bit exists as the 30th bit, e.g. `1` would
become `536870913` (`(1 << 29) | 1`).

Newly defined inv types are as follows:

- `EXT_TX`: `536870913` (`(1 << 29) | 1`)
- `EXT_BLOCK`: `536870914` (`(1 << 29) | 2`)

#### Backward Compatability

To avoid an enormous amount of orphan transactions on non-upgraded nodes,
upgraded nodes shall respond with a NOTFOUND message in response to any regular
GETDATA TX request which maps to an extension chain transaction. Invs
containing extension transactions shall not be broadcast to non-upgraded peers.

Responses GETDATA EXT_TX shall include both extension and non-extension
transactions.

### `EXT_BLOCK` message serialization

`block` messages requested with the `EXT_BLOCK` inv type are to use the
canonical serialization with an extra varint count appended. Following the
varint count shall be a transaction vector using BIP141 serialization. Clients
MUST disregard blocks which use BIP141 serialization in the canonical
transaction vector.

TODO: Expand.

### Extensions to Compact Block Relay (BIP152)

Compact block relay shall be initiated using the previously specified (BIP152)
`sendcmpct` message, with a version of `3`. Serialization for `cmpctblock`,
`blocktxnrequest`, and `blocktxn` are modified to include two transaction
vectors. The first being canonical, and the second being extended.

TODO: Expand.
