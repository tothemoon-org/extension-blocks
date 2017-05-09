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

The extension block shall be delivered in the same payload as a canonical block
in the case of a `GETDATA` message with an inv type of `WITNESS_BLOCK`.

#### Backward Compatability

To avoid an enormous amount of orphan transactions on non-upgraded nodes,
upgraded nodes shall respond with a `NOTFOUND` message in response to any
regular `GETDATA TX` request which maps to an extension chain transaction. Invs
containing extension transactions shall not be broadcast to non-upgraded peers.

Responses to a `GETDATA` message with an inv type of `WITNESS_TX` shall include
both extension and non-extension transactions.

### Extra `WITNESS_BLOCK` message serialization

`block` messages requested with the `EXT_BLOCK` inv type are to use the
canonical serialization with an extra varint count appended. Following the
varint count shall be a transaction vector using BIP141 serialization. Clients
MUST disregard blocks which use BIP141 serialization in the canonical
transaction vector.

In other words, extension block payloads are combined with a canonical block
payload, in the form of:

```
[80-byte header]
[varint tx count]
[tx vector]
[varint extension tx count]
[extension tx vector]
```

### Extensions to Compact Block Relay (BIP152)

TODO.

## Reference Implementation

https://github.com/bcoin-org/bcoin-extension-blocks

## Copyright

This document is hereby placed in the public domain.
