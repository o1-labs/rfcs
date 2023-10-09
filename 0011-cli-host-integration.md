---
title: Client/Host integration
---


# Abstract

This document presents the architecture of the integration between the
VM, the program it runs (called the *client*), and the host.

The main requirement of this layer in the zkVM implementation is to be
fully compatible with the current protocol used for fault proofs in
Optimism, more specifically the protocol that allows interactions
between Cannon et the `op-program` host.

Here is a sketch of the interactions between the aforementioned trio of
client/VM/host.

``` text

+-----------------+           +----------------+         +----------------+
|                 |           |                |         |                |
|                 |           |                |         |                |
|     Client      o<--------->o     zkVM       o<------->o      Host      |
|     (MIPS)      |           |                |         |  (op-program)  |
|                 |           |                |         |                |
+-----------------+           +----------------+         +----------------+
```

The VM sits right in the middle and must propagate to the host the
client requests for data (say from the L1/L2) and decode the responses
from the host to get the data back to the client program.

In the context of `Optimism`, the targeted host is `op-program` and the
related client `op-program-client`.

# Types of data

Data exchange is focused on 2 types of data, whose specifications we
detail below.

preimage
:   the so-called \"pre-image oracle\" is the **only** way that a
    running program can communicate with the virtual machine it is
    running on. This is how the program can query input data (initial
    inputs, external data). See [#preimage](#preimage).

hints
:   hints denote application-specific way of informing that some data
    will be requested shortly so as to prepare their availability. See
    [#hints](#hints)

# Communication protocols

Data exchange between the client and the VM is done via 2 communication
protocols called **pre-image** and **hinting** protocols. We recall here
their specifications.

## Pre-image communication

Implementing the pre-image protocol is **required** to be able to
communicate with the current hosts.

### Request format

A pre-image request is a **32** bytes long value, decomposed in 2 parts
as follows:

-   a [key type](#key_types) (1 byte) to denote the kind of key we are
    asking the preimage of;
-   the key itself (31 bytes)

``` {#preimage_request_format .text}
+--------------+-----------+
| key type <1> | key <31>  |
+--------------+-----------+
```

### Key types

Since key types are represented by a single byte, they can have 256
values, with predefined meanings. See [key types
specifications](https://github.com/ethereum-optimism/optimism/blob/develop/specs/fault-proof.md#pre-image-key-types).

  Value      Key type
  ---------- -----------------------------------------------------------------------
  0          Key zero (should not be used)
  1          Local key
  2          Global Keccak256 key
  3          Global generic key (reserved)
  4..128     Reserved range (for future usage)
  129..255   Application specific (may be used by forks or customized fault proof)

As a first approximation at the moment, we can consider that only key
values `1` and `2` should ever occur.

Note that a 32-bytes Kecca256 key will be encoded by substituting the
first byte of the actual key with the Global Kecca256 key type, and
keeping the remaining 31 bytes intact. The assumption here is that there
is enough entropy in the remaining 31 bytes for this substitution not to
be impactful.

### Response format

The response, of variable length, consists of a byte stream with a
8-bytes (`uint64`) length prefix together with `length`-bytes pre-image
data.

``` {#preimage_response_format .text}
+------------+--------------------+
| length <8> | pre-image <length> |
+---------------------------------+
```

## Hints communication

Supporting the hint protocol is marked as **optional** in the reference
documentation. However, existing examples **always** use it before a
given pre-image request.

As stated in
[hinting](https://github.com/ethereum-optimism/optimism/blob/develop/specs/fault-proof.md#hinting):

> Hinting is implemented with a request-acknowledgement wire-protocol
> over a blocking two-way stream

### Request format

A hint is a byte stream where the first 8-bytes encode a `uint64` length
parameter, followed by `<length>` hint bytes.

``` {#hint_request_format .text}
+------------+---------------+
| length <8> | hint <length> |
+----------------------------+
```

### Hint types

Hint types are application-specific. However, since the goal is to able
to be a drop-in replacement for the op-program we need support the
following hint types.

  -----------------
  l1-block-header
  l1-transactions
  l1-receipts
  l2-block-header
  l2-transactions
  l2-code
  l2-state-node
  l2-output
  -----------------

More details regarding these requests can be obtained directly from the
[pre-image-hinting-routes](https://github.com/ethereum-optimism/optimism/blob/develop/specs/fault-proof.md#pre-image-hinting-routes)
section.

### Response format

Hint responses are a single `ack` byte that informs the client that the
hint has been processed.

``` {#hint_response_format .text}
+---------+
| ack <1> |
+---------+
```

# Conclusion

The [pre-image](#preimage) and [hints](#hints) communication protocols
are simple byte-level exchange protocols that we need to implement in
this order for one is required and the other (officially) optional.

On that note, let us quote the online documentation, that states:

> Servers may respond to hints and pre-image (see below) requests
> asynchronously as they are on separate streams. To avoid requesting
> pre-images that are not yet fetched, clients should request the
> pre-image only after it has observed the hint acknowledgement.

# Resources

-   The main [fault-proof
    documentation](https://github.com/ethereum-optimism/optimism/blob/develop/specs/fault-proof.md)
-   [L1
    hints](https://github.com/ethereum-optimism/optimism/blob/develop/op-program/client/l1/hints.go),
    [L2
    hints](https://github.com/ethereum-optimism/optimism/blob/develop/op-program/client/l2/hints.go)
-   `op-program`
    [client](https://github.com/ethereum-optimism/optimism/blob/develop/op-program/client)
    &
    [host](https://github.com/ethereum-optimism/optimism/blob/develop/op-program/host)
