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
`1` for ok responses.

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
### Common Admin Types

These types are used in multiple admin messages.

#### PeerID Type

```c++
  typedef uint64_t peerid_type;
```

Just a random uint thats created the first tine the daemon runs and is stored in p2pstate.bin

#### Basic Node Data

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

To see how basic_node_data is handled look [here]() 

#### Core Sync Data

```c++
struct CORE_SYNC_DATA
  {
    uint64_t current_height; // the current height of the peers blockchain
    uint64_t cumulative_difficulty; // lower 64 bits of the peers blockchains cumulative_difficulty 
    uint64_t cumulative_difficulty_top64; // upper 64 bits of the peers blockchains cumulative_difficulty 
    crypto::hash  top_id; // The hash of the most recent block in the peers blockchain
    uint8_t top_version; // The major version of the peers top block 
    uint32_t pruning_seed; // The peers pruning seed
  }
```

To see how Core Sync Data is handled look [here]()

#### Peer List Entry Base

```c++
  struct peerlist_entry_base
  {
    AddressType adr; // The address IPv(4|6)/ I2p/ Tor
    peerid_type id; // A random u64 same as in basic_node_data 
    int64_t last_seen; // The last time we recived a message from the peer will be set 0 when sending known peers to peers
    uint32_t pruning_seed; // The peers pruning seed
    uint16_t rpc_port; // The peers rpc port
    uint32_t rpc_credits_per_hash; // The peers rpc credits per hash - used for charging for rpc
  }
```

Peer list entry base is used for every white/ grey peer in Monero's peer list, It stores information needed to decide optimal peers to connect to. 

### Handshake (`1001`)

Used to establish a P2P connection with a peer. 

<details>

<summary>Request</summary>

A handshake request currently looks like:

```c++
struct request_t
    {
      basic_node_data node_data;
      t_playload_type payload_data;
    };
```
[basic_node_data](#basic-node-data)

[payload_data](#core-sync-data) *payload data is just core_sync_data

To see how a handshake request is handled look [here]()

</details>

<details>

<summary>Response</summary>
A Handshake response currently looks like:

```c++
struct response_t
    {
      basic_node_data node_data;
      t_playload_type payload_data;
      std::vector<peerlist_entry> local_peerlist_new;
    }
```

[basic_node_data](#basic-node-data)

[payload_data](#core-sync-data) *payload data is just core_sync_data

peerlist_entry being:

```c++
  typedef peerlist_entry_base<epee::net_utils::network_address> peerlist_entry;
```
[Peer List Entry Base](#peer-list-entry-base)

To see how a handshake response is handled look [here]()

</details>

### Timed Sync (`1002`)

Used when the peer hasn't sent a message in a while or to retrieve peers from the peer.

<details>
<summary>Request</summary>
A timed sync request currently looks like:

```c++ 
struct request_t
    {
      t_playload_type payload_data;
    }
```

[payload_data](#core-sync-data) *payload data is just core_sync_data

To see how a timed sync request is handled look [here]() 

</details>

<details>
<summary>Response</summary>

```c++
struct response_t
{
  t_playload_type payload_data;
  std::vector<peerlist_entry> local_peerlist_new;
}
```
[payload_data](#core-sync-data) *payload data is just core_sync_data

peerlist_entry being:

```c++
  typedef peerlist_entry_base<epee::net_utils::network_address> peerlist_entry;
```
[Peer List Entry Base](#peer-list-entry-base)

To see how a timed sync response is handled look [here]()

</details>


### Ping (`1003`)
Used to make "callback" connection, to be sure that opponent node 
have accessible connection point. Only other nodes can add peer to peerlist,
and ONLY in case when peer has accepted connection and answered to ping.

<details>
<summary>Request</summary>

```c++
struct request_t
{
  /*actually we don't need to send any real data*/
};
```
To see how a ping request is handled look [here]()

</details>

<details>
<summary>Response</summary>

```c++
struct response_t
{
  std::string status;
  peerid_type peer_id; 
};
```

[peer_id](#peerid-type)

To see how a ping response is handled look [here]()


</details>

### Support Flags (`1007`)

Used when the peer does not send their support flags in the handshake.
<details>
<summary>Request</summary>

```c++
struct request_t
    {
      // Again don't actually need to send any data  
    };
```

To see how a support flags request is handled look [here]()


</details>

<details>
<summary>Response</summary>

```c++
struct response_t
{
  uint32_t support_flags; // used to show the features this node supports the only current feature is fluffy blocks
}
```

To see how a support flags response is handled look [here]()


</details>

## Cryptonote Protocol Commands

### Common Protocol Types

#### TX Blob Entry

```c++
struct tx_blob_entry
{
  blobdata blob; // The Tx Blob pruned or non-pruned
  crypto::hash prunable_hash; // The prunable hash this field wont exist for a non-pruned tx
}
``` 


#### Block Complete Entry

```c++

struct block_complete_entry
{
  bool pruned; // If this block is pruned or not
  blobdata block; // The block blob
  uint64_t block_weight; // The blocks weight will be 0 for a non-pruned block
  std::vector<tx_blob_entry> txs; // The block transactions
}
```

[Tx Blob entry](#tx-blob-entry)


### New Block (`2001`)

Used before fluffy blocks to notify peers of a new block.

<details>
<summary>Details</summary>

```c++
struct request_t
{
  block_complete_entry b;
  uint64_t current_blockchain_height; // The height of the peers Blockchain
};
```

[Block Complete Entry](#block-complete-entry) - This will contain ALL the transaction un-pruned.

To see how a New Block is handled look [here]()

</details>

### New Transactions (`2002`)

Used to notify peers of new tx-pool transactions.

<details>
<summary>Details</summary>

```c++
struct request_t
{
  std::vector<blobdata>   txs; // A list of un-pruned tx blobs
  std::string _; // padding
  bool dandelionpp_fluff; //true if this is a dpp fluff
};
```

To see how a New Block is handled look [here]()

</details>

### Request Get Objects (`2003`)

Used to fetch Blocks by hash from a peer.  

<details>
<summary>Details</summary>


```c++
struct request_t
{
  std::vector<crypto::hash> blocks; // The block ids
  bool prune; // If we mind the blocks being pruned 
}
```

To see how a request get objects is handled look [here]()

</details>


### Response Get Objects (`2004`)

A response to a request get objects 

<details>
<summary>Details</summary>

```c++
struct request_t
{
  std::vector<block_complete_entry>  blocks; // Blocks with all the txs
  std::vector<crypto::hash>          missed_ids; // blocks we did not have
  uint64_t                         current_blockchain_height;
};
```

[Block Complete Entry](#block-complete-entry)

To see how a response get objects is handled look [here]()


</details>


### Request Chain (`2006`)
Request to get the peers block ids that we are missing.

note the amount of Block ids that can be sent in one response is capped. 

<details>
<summary>Details</summary>

```c++
struct request_t
{
  std::list<crypto::hash> block_ids; /*IDs of the first 10 blocks are sequential, next goes with pow(2,n) offset, like 2, 4, 8, 16, 32, 64 and so on, and the last one is always genesis block */
  bool prune;
}
```

To see how a request chain entry is handled look [here]()


</details>

### Response Chain Entry (`2007`)

A response to a request chain.

<details>
<summary>Details</summary>

```c++
struct request_t
{
  uint64_t start_height; // the height of the first block id
  uint64_t total_height; // the hieght of the peers chain
  uint64_t cumulative_difficulty; 
  uint64_t cumulative_difficulty_top64;
  std::vector<crypto::hash> m_block_ids; // the block ids
  std::vector<uint64_t> m_block_weights; 
  cryptonote::blobdata first_block; // the first block where the peers and our chain potentailly diverge will so will be m_block_ids[1]
}
```
To see how a response chain entry is handled look [here]()


</details>


### New Fluffy Block (`2008`)

A notification of a new block. Sent initally without any transactions but could come in response to a [Request Fluffy Missing TX](#request-fluffy-missing-tx-2009) in which case 
this will contain the requested missing txs.

<details>
<summary>Details</summary>

```c++
struct request_t
{
  block_complete_entry b;
  uint64_t current_blockchain_height;
}
```

[Block Complete Entry](#block-complete-entry)


To see how a new fluffy block is is handled look [here]()


</details>

### Request Fluffy Missing TX (`2009`)

A request for txs we are missing from an incomming fluffy block. 

<details>
<summary>Details</summary>

```c++
struct request_t
{
  crypto::hash block_hash; // The block we wan't the txs from
  uint64_t current_blockchain_height; 
  std::vector<uint64_t> missing_tx_indices; // the idxs of the transactions in the block
}
```

#### missing_tx_indices

This is literally just the idx of the transaction in the block so if we are missing the first tx in the block we would ask for tx at idx 0.

To see how a request fluffy missing tx is is handled look [here]()


</details>


### Get Tx Pool Compliment (`2010`)

A request for tx pool transactions we don't have

<details>
<summary>Details</summary>


```c++
struct request_t
{
  std::vector<crypto::hash> hashes; // our tx-pool transaction hashes 
}
```

</details>