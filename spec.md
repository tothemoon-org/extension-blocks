# Extension Blocks

```
Layer: Consensus (soft-fork)
Title: Extension Blocks
Author: Christopher Jeffrey <chjj@purse.io>
        Joseph Poon <joseph@lightning.network>
        Fedor Indutny <fedor@indutny.com>
        Stephen Pair <stephen@bitpay.com>
Status: Draft
Created: 2017-03-17
License: Public Domain
```

## Abstract

This specification defines a method of increasing bitcoin transaction
throughput without altering any existing consensus rules.

## Motivation

The bitcoin network's transaction throughput is correlated with its consensus
rules regarding retargetting and denial-of-service limits.

Bitcoin retargetting ensures that the time in between mined blocks will be
roughly 10 minutes. It is not possible to change this rule. There has been
debate regarding other ways of greatly increasing transaction throughput, with
no proposed consensus-layer solutions that have proven themselves to be
particularly safe.

## History

_Auxiliary blocks_, as first proposed by [Johnson Lau in 2013][aux], outlined a
way of having funds enter and exit an additional block by using special
opcodes.

_Extension blocks_ were a later proposal, also [from Lau in 2017][ext], which
revised many of the ideas of the original proposal.

## Specification

Extension blocks devise a second layer on top of canonical bitcoin blocks in
which a miner will commit to the merkle root of an additional block of
transactions.

Extension blocks leverage several features of BIP141, BIP143, and BIP144 for
transaction opt-in, serialization, verification, and network services. This
specification should be considered an extension and modification to these BIPs.
Extension blocks are _not_ compatible with BIP141 in its current form, and will
require a few minor additional rules.

Extension blocks maintain their own UTXO set in order to avoid interference
with the existing UTXO set of non-upgraded nodes. This is a tricky endeavor,
requiring a `resolution` transaction to be present at the end of every
canonical block.

This specification prescribes a way of fooling non-upgraded nodes into
believing the existing UTXO set is still behaving as they would expect.
Unfortunately, this requires a bit of extra book-keeping on the upgraded node's
side.

### Commitment

An upgraded miner willing to include extension block is to include an extra
coinbase output of 0 value. The output script exists as such:

```
OP_RETURN 0x24 0xaa21a9ef[32-byte-merkle-root]
```

The commitment serialization and discovery rules follows the same rules defined
in BIP141.

The merkle root is to be calculated as a merkle tree with all extension and
canonical block txids and wtxids as the leaves.

Any block containing an extension block MUST include an extension commitment
output.

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

Every block containing entering or exiting outputs MUST contain a final
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

Fees are to propagate up from the extension block into the resolution
transaction. In other words, the resolution transaction's fee should should be
equal to the amount of total fees collected in the extension block.

#### Bootstrapping

In order to bootstrap the activation of extension blocks, a "genesis"
resolution transaction MUST be mined in the first block to include any entering
outputs (the `activation` block). This is the only resolution transaction in
existence that does not require a reference to a previous resolution
transaction.

The genesis resolution transaction MAY also include a 1-100 byte script in the
first input, containing a single push-only opcode. This allows the miner of the
genesis resolution to add a special message. The input script MUST execute
without failure (no malformed pushdatas, no OP_RESERVED).

#### Resolution Rules

The resolution transaction's first output MUST have a value equal to:

`(previous-resolution-value + entering-value - exiting-value) - ext-block-fees`

The following output scripts and values must replicate the extension block's
exiting outputs _exactly_ (in the same order they appear in the ext. block).

The resolution transaction's version MUST be set to the uint32 max (`2^32 - 1`).
After activation, this version is forbidden by consensus rules to be used with
any other transaction on the canonical chain or extension chain. This is
required for easy non-contextual identification of a resolution transaction.

### Entering the extension block

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

### Exiting the extension block

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
  - Script: P2PKH (duplicated from the exited output below)
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

#### Exit Redemption

As described above, outputs exit from the extension block onto the main chain
by way of the resolution transaction. The outpoint created in the extension
block MUST not to be spent on either chain. Instead, exiting outputs must be
spent from outpoints created on the resolution transaction.

#### Exit Maturity Requirement

Similar to coinbase transactions, resolution transactions can also be
permanently un-mined in the case of a reorganization. This potentially
invalidates all spends from any exiting outputs it may have contained,
rendering the spending transactions not relayable and no longer mineable on the
best chain. An exit maturity requirement is required for this reason.

See: https://github.com/tothemoon-org/extension-blocks/issues/9

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

### Verification

Verification of transactions within the extension block shall enforce all
currently deployed softforks, along with an extra BIP141-like ruleset.

Transactions within the extended transaction vector MAY include a witness
vector using BIP141 transaction serialization.

Verification shall be performed on extended transactions with `VERIFY_WITNESS`
rules.

Extended transactions MUST NOT have any access to the canonical UTXO set.

If an extension block fails any consensus check, the upgraded node MUST
consider the entire block invalid.

### BIP141 Rule Changes

- Aside from the resolution transaction, witness program outputs are only
  redeemable inside of an extension block.
- Witness transactions may _only_ contain witness program inputs.
- BIP141's nested P2SH feature is no longer available, and no longer a
  consensus rule.
- The concepts of `block weight` and `transaction weight` have been removed.
- The concept of `sigops cost` remains present for future soft-forkable and
  upgradeable DoS limits.

### DoS Limits

DoS limits shall be enforced by extension block size as well as the newly
defined metrics of inputs and outputs cost. Be aware that exiting outputs
inside the extension block affect DoS limits in the canonical block, as they
add size and legacy sigops.

```
MAX_BLOCK_SIZE: 1000000 (unchanged)
MAX_BLOCK_SIGOPS: 20000 (unchanged)
MAX_EXTENSION_SIZE: 6000000
MAX_EXTENSION_COST: 9600000
MAX_EXTENSION_SIGOPS_COST: 320000
```

The maximum extension size is intentionally high (6mb). The average case block
is truly limited by "extension cost" and "sigops cost".

The max extension cost will result in an average case block size of roughly
2mb, with a worst case of 6mb (assuming a miner includes garbage data in
witness vectors).

Extension blocks leverage BIP141's upgradeable script behavior to also allow
for upgradeable DoS limits. Future size and computational scalability can be
soft-forked in with the addition of new witness programs. On non-upgraded
nodes, unknown witness programs count as an extension cost factor of 1. Lesser
cost can be implemented for newer witness programs in order to allow future
soft-forked dos limit changes.

#### Calculating Inputs Cost

Witness programs with a version of zero shall be worth 8 points for every byte
in the serialized witness vector (including the varint item count).

Unknown witness programs shall be worth 1 point for every byte in the witness
vector.

#### Calculating Outputs Cost

Witness programs with a version of zero shall be worth 8 points for every byte
in the output script (including the varint size prefix).

Exiting outputs shall be worth 8 points for every byte in the output script.
Note that this cost is not upgradeable.

Unknown witness programs shall be worth 1 point for every byte in output script.

#### Calculating Sigops Cost

Witness key hash v0 shall be worth 8 points.

Witness script hash v0 shall be worth the number of accurately counted sigops
(bip16 counting) in the redeem script, multiplied by a factor of 8.

#### Dust Threshold

A consensus dust threshold is now enforced within the extension block.

Outputs containing less than 500 satoshis of value are _invalid_ within an
extension block. This _includes_ entering outputs as well, but not exiting
outputs.

### Rules for extra Lightning security

Lightning Network faces certain kinds of systemic attacks as defined in the
Lightning Network whitepaper risks (mass-closeout of old states).

If the second highest transaction version bit (30th bit) is set to to `1`
within an extension block transaction, an extra 700-bytes is reserved on the
transaction space used up in the block. [NOTE: Transaction space and sigops
cost not yet defined]

This space per transaction is pre-allocated and can be consumed in the same
block by two transactions (of a maximum size of 350 bytes each), which fulfill
specific constraints as defined below.

The first allocation may only be consumed by a transaction which directly
spends from the first output of the transaction with the 30th bit set.

The second allocation may only consume the first output of ANY transaction
within the past 2016 blocks which have the 30th bit set.

If the allocation is not used by other transactions, the transaction consumes
that extra space, reducing the blocksize by up to 700 bytes in available space.

This is a consensus rule within the extension block and does not apply to the
main-chain block.

The purpose of this is to ensure that without miner coordination, the costs
will be unusually high for systemic attacks, since blockspace is preallocated
for miners to include penalty Lightning Network transactions, and this is
enforced since both parties must agree to the version bit in Commitment
Transaction in Lightning. This is an opt-in feature in Lightning, and trades
higher transaction fees for greater availability for penalties. The assumption
is that in the majority of cases of an incorrect broadcast, the penalty will be
included in the same block via the second allocation, and give room for other
transactions in the first allocation.

### Migration and adoption concerns

Most of the bitcoin ecosystem is currently well-equipped to handle an upgrade
to BIP141.

For wallets currently supporting BIP141, the migration should be trivial.

For fullnode-based services, APIs may be altered to transparently serve
extension block transactions to clients as if they appeared in the canonical
block. This, of course, would not include any miner API.

### Wallet concerns and migration

Wallets currently supporting BIP141 must be modified in a few key ways in order
to achieve compatibility with extension blocks.

1. Wallets must pick a chain to spend from when creating a transaction (either
   the canonical block or the extension block, but not both). In other words,
   transactions must have all witness program inputs, or all non-witness program
   inputs. For wallets that support both chains, the coin selector can
   automatically pick which chain to use if the user does not specify.

2. Wallets supporting extension blocks must ignore inputs on
   resolution transactions if seen. This should be a simple check of transaction
   version number, and similar to how wallets already ignore a coinbase's input.
   This is necessary to prevent wallets from mistakenly seeing a double spend.

3. Wallets supporting both canonical block and extension block funds
   must ignore exiting outputs within the extension block. This is necessary to
   prevent wallets from mistakenly indexing the same output twice.

The latter two points only apply to wallets with operate via direct blockchain
monitoring. Monitoring wallets typically watch the blockchain and index their
own transactions and outputs.

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

### Data Migration Concerns

It is likely that implementations will need to include an extra bit on every
stored UTXO in order to mark which chain they belong to. Depending on the
implementation's UTXO serialization/compression format, this may require a
database migration.

### Activation

- Version bit: 2
- Deployment name: `extblk` (appears as `!extblk` in GBT).
- Start time: TBD
- Timeout: TBD

### Deactivation

Miners may vote on deactivation of the extension block via the 28th BIP9
softfork bit. The "deactivation" deployment's start time shall begin 3 years
after the start time of extension blocks, with a miner activation threshold of
95%. The minimum locked-in period must be at least 26 retarget intervals (1
year).

By this point, a future extension block ruleset will likely have been
developed, which is superior in terms of feature-set and scalability (see also:
[Rootstock][rsk] and/or [Mimblewimble][mw]). This enables updates for long-term
scalability solutions with minimal baggage of supporting deprecated chains.

After deactivation, it is expected by the community that exits from the
extension block will still still be possible and secure according to the terms
of the yet-to-be-designed soft-fork, but entrance into and transfer of funds
within the extension block could be no longer permitted.

We propose two possible solutions for deactivation, one of which will be
selected in the final version of this document.

#### Deactivation Option #1

Upon activation of the 28th bit, the resolution output will return to being an
output which anyone-can-spend as a consensus rule today. This 28th BIP9 bit (or
another BIP9 bit in conjunction) can be overloaded to enable soft-fork
activation to prevent this from actually being an anyone-can-spend in the
future. This allows for enabling future extension block features without
hard-forking the code.

A social contract is understood whereby the funds in the extension block will
be usable and redeemable in the general deactivation design below. If proper
and safe withdrawals are not activated within the terms, users and exchanges
can refuse to acknolwedge blocks with the bit set as a soft-fork.

It is possible to do a direct upgrade of the extension block by using the same
output set upon BIP9 activation of the 28th bit in conjunction with new rules
on another bit (e.g. 27th bit activates the new chain conditionally upon the
28th bit also being activated, creating a direct migration into a new extension
block without new transactions on the main-chain).

It is understood that this soft-fork will be overloaded with enforcement of
withdrawal/redemption of funds so that it requires the script terms to be
parsed upon withdrawal.

#### Deactivation Option #2

Upon activation of the 28th bit, no further transactions are permitted inside
the extension block. Withdrawals to the main-chain only via merkle proofs are
only permitted.

This requires code and specification for merkle-proof withdrawals to be
specified and available today.

#### Deactivation via Merkle Tree Proofs

Redemption from the old extension block to the main-chain and new extension
block can be migrated by way of merkle proofs to be designed in the future.
Funds may be imported to the new extension block by hard-coding a UTXO merkle
root into the implementation as a consensus rule, and verifying imported funds
against a merkle path.

To enable importing, nodes require only a copy of the current 32-byte merkle
root of the deactivated extension block.

This removes the necessity for full nodes to store a copy of the full UTXO set
on disk, with a tradeoff of larger on-chain transactions when redeeming in the
future. An alternative would be for all clients to maintain a record of the
UTXO set and keep the full bitfield in memory.

Since the TXO set is static (with only whether it is unspent or spent changing)
as it is deactivated from new outputs, this is simpler than currently proposed
designs for changing UTXOs:
https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2015-October/011638.html

#### Motivation

While deactivation may be seen as a drawback, the primary benefit of this
specification is _not_ an extension block with a BIP141-ruleset. Instead, it is
that the ecosystem will have laid the groundwork for handling extension blocks.
The future of the bitcoin protocol may include several different extension
blocks with many different rulesets.

## Reference Implementation (WIP)

https://github.com/bcoin-org/bcoin-extension-blocks

## Copyright

This document is hereby placed in the public domain.

[aux]: https://bitcointalk.org/index.php?topic=283746.0
[ext]: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-January/013490.html
[rsk]: https://uploads.strikinglycdn.com/files/90847694-70f0-4668-ba7f-dd0c6b0b00a1/RootstockWhitePaperv9-Overview.pdf
[mw]: https://scalingbitcoin.org/papers/mimblewimble.txt
