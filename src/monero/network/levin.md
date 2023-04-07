# Levin

> This file has mostly been ripped from Monero

This is a document explaining the current design of the Levin protocol, as
used by Monero. The protocol is largely inherited from cryptonote, but has
undergone some changes.

This document also may differ from the `struct bucket_head2` in Monero's
code slightly - the spec here is slightly more strict to allow for
extensibility.

## Contents

- [Levin Headers](#header)
- [Message Flow](#message-flow)
- [Commands](#commands)
    - [Admin](#p2p-admin-commands)
        - [Handshake](#handshake-1001)
        - [Timed Sync](#timed-sync-1002)
        - [Ping](#ping-1003)
        - [Support Flags](#support-flags-1007)
    - [Protocol](#cryptonote-protocol-commands)
        - [New Block](#new-block-2001)
        - [New Transactions](#new-transactions-2002)
        - [Request Get Objects](#request-get-objects-2003)
        - [Response Get Objects](#response-get-objects-2004)
        - [Request Chain](#request-chain-2006)
        - [Response Chain Entry](#response-chain-entry-2007)
        - [New Fluffy Block](#new-fluffy-block-2008)
        - [Request Fluffy Missing TX](#request-fluffy-missing-tx-2009)
        - [Get Tx Pool Compliment](#get-tx-pool-compliment-2010)


# Header
This header is sent for every Monero p2p message.


```
 0               1               2               3
 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      0x01     |      0x21     |      0x01     |      0x01     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      0x01     |      0x01     |      0x01     |      0x01     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Length                            |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  E. Response  |                   Command
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                |                 Return Code
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                |Q|S|B|E|               Reserved
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                |      0x01     |      0x00     |      0x00     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     0x00      |
+-+-+-+-+-+-+-+-+
```
```c++
  struct bucket_head2
  {
    uint64_t m_signature;
    // length
    uint64_t m_cb;
    uint8_t  m_have_to_return_data;
    uint32_t m_command;
    int32_t  m_return_code;
    uint32_t m_flags;
    uint32_t m_protocol_version;
  };

```


### Signature

The first 8 bytes are the "signature" which helps identify the protocol (in
case someone connected to the wrong port, etc). The comments indicate that byte
sequence is from "benders nightmare".

This also can be used by deep packet inspection (DPI) engines to identify
Monero when the link is not encrypted. SSL has been proposed as a means to
mitigate this issue, but BIP-151 or the Noise protocol should also be considered.

### Length

The length is an unsigned 64-bit little endian integer. The length does _not_
include the header.

The implementation currently rejects received messages that exceed 100 MB
(base 10) by default.

### Expect Response

A zero-byte if no response is expected from the peer, and a non-zero byte if a
response is expected from the peer. Peers must respond to requests with this
flag in the same order that they were received, however, other messages can be
sent between responses.

There are some commands in the
[cryptonote protocol](#cryptonote-protocol-commands) where a response is
expected from the peer, but this flag is not set. Those responses are returned
as notify messages and can be sent in any order by the peer.

### Command

An unsigned 32-bit little endian integer representing the Monero specific
command being invoked.

### Return Code

A signed 32-bit little endian integer representing the response from the peer
from the last command that was invoked. This is `0` for request messages and
1 for ok responses.

```c++
#define LEVIN_OK                                        0
#define LEVIN_ERROR_CONNECTION                         -1
#define LEVIN_ERROR_CONNECTION_NOT_FOUND               -2
#define LEVIN_ERROR_CONNECTION_DESTROYED               -3
#define LEVIN_ERROR_CONNECTION_TIMEDOUT                -4
#define LEVIN_ERROR_CONNECTION_NO_DUPLEX_PROTOCOL      -5
#define LEVIN_ERROR_CONNECTION_HANDLER_NOT_DEFINED     -6
#define LEVIN_ERROR_FORMAT                             -7

```

### Flags

 * `Q` - Bit is set if the message is a request.
 * `S` - Bit is set if the message is a response.
 * `B` - Bit is set if this is a the beginning of a [fragmented message](#fragmented-messages).
 * `E` - Bit is set if this is the end of a [fragmented message](#fragmented-messages).

### Version

A fixed value of `1` as an unsigned 32-bit little endian integer.


# Message Flow

The protocol can be subdivided into: (1) notifications, (2) requests,
(3) responses, (4) fragmented messages, and (5) dummy messages. Response
messages must be sent in the same order that a peer issued a request message.
A peer does not have to send a response immediately following a request - any
other message type can be sent instead.

## Notifications

Notifications are one-way messages that can be sent at any time without
an expectation of a response from the peer. The `Q` bit must be set, the `S`,
`B` and `E` bits must be unset, and the `Expect Response` field must be zeroed.

Some notifications must be in response to other notifications. This is not
part of the levin messaging layer, and is described in the
[commands](#commands) section.

## Requests

Requests are the basis of the admin protocol for Monero. The `Q` bit must be
set, the `S`, `B` and `E` bits must be unset, and the `Expect Response` field
must be non-zero. The peer is expected to send a response message with the same
`command` number.

## Responses

Response message can only be sent after a peer first issues a request message.
Responses must have the `S` bit set, the `Q`, `B` and `E` bits unset, and have
a zeroed `Expect Response` field. The `Command` field must be the same value
that was sent in the request message. The `Return Code` is specific to the
`Command` being issued (see [commands])(#commands)).

## Fragmented

Fragmented messages were introduced for the "white noise" feature for i2p/tor.
A transaction can be sent in fragments to conceal when "real" data is being
sent instead of dummy messages. Only one fragmented message can be sent at a
time, and bits `B` and `E` are never set at the same time
(see [dummy messages](#dummy)). The re-constructed message must contain a
levin header for a different (non-fragment) message type.

The `Q` and `S` bits are never set and the `Expect Response` field must always
be zero. The first fragment has the `B` bit set, neither `B` nor `E` is set for
"middle" fragments, and `E` is set for the last fragment.

## Dummy

Dummy messages have the `B` and `E` bits set, the `Q` and `S` bits unset, and
the `Expect Response` field zeroed. When a message of this type is received, the
contents can be safely ignored.


# Commands

## P2P (Admin) Commands

Thses commands are in a request/ response format. Peers are expected to return 
responses in the order they are recived. 

### Handshake (`1001`)

#### Request

A handshake request currently looks:

```c++
struct request_t
    {
      basic_node_data node_data;
      t_playload_type payload_data;
    };
```

with basic_node_data being:

```c++
struct basic_node_data
  {
    uuid network_id; // An identifyier for the network the peer is on (MAINNET | TESTNET | STAGENET)                 
    uint32_t my_port; // The peers p2p port
    uint16_t rpc_port; // The peers rpc port
    uint32_t rpc_credits_per_hash; // used for "charging" people to use your node although no one uses it
    peerid_type peer_id; // just a random u64 which is created when the node is started for the first time and stored in the p2pstate.bin
    uint32_t support_flags; // used to show the features this node supports the only current feature is fluffy blocks
  }
```
and payload_data being:

```c++
struct CORE_SYNC_DATA
  {
    uint64_t current_height;
    uint64_t cumulative_difficulty;
    uint64_t cumulative_difficulty_top64;
    crypto::hash  top_id;
    uint8_t top_version;
    uint32_t pruning_seed;
  }
```

### Timed Sync (`1002`)
### Ping (`1003`)
### Support Flags (`1007`)

## Cryptonote Protocol Commands

### New Block (`2001`)
### New Transactions (`2002`)
### Request Get Objects (`2003`)
### Response Get Objects (`2004`)
### Request Chain (`2006`)
### Response Chain Entry (`2007`)
### New Fluffy Block (`2008`)
### Request Fluffy Missing TX (`2009`)
### Get Tx Pool Compliment (`2010`)

