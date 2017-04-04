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

This specification defines peer services for extension blocks.

## Specification

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

### Extensions to Compact Block Relay (BIP152)

Compact block relay shall be initiated using the previously specified (BIP152)
`sendcmpct` message, with a version of `3`. Serialization for `cmpctblock`,
`blocktxnrequest`, and `blocktxn` are modified to include two transaction
vectors. The first being canonical, and the second being extended.

## Reference Implementation

https://github.com/bcoin-org/bcoin-extension-blocks

## Copyright

This document is hereby placed in the public domain.
