# Admin Protocol

The Monero admin protocol contains the logic for connecting to peers and making sure peers are staying active.

## Network ID

Monero has IDs for it's different networks which are sent in handshakes to make sure we are connecting to peers on the same network.
The IDs are:

#### Mainnet
```c++
const NETWORK_ID = { {
      0x12 ,0x30, 0xF1, 0x71 , 0x61, 0x04 , 0x41, 0x61, 0x17, 0x31, 0x00, 0x82, 0x16, 0xA1, 0xA1, 0x10
    } }; // Bender's nightmare

```
#### Testnet
```c++
const NETWORK_ID = { {
        0x12 ,0x30, 0xF1, 0x71 , 0x61, 0x04 , 0x41, 0x61, 0x17, 0x31, 0x00, 0x82, 0x16, 0xA1, 0xA1, 0x11
      } }; // Bender's daydream

```
#### Stagenet
```c++
const NETWORK_ID = { {
        0x12 ,0x30, 0xF1, 0x71 , 0x61, 0x04 , 0x41, 0x61, 0x17, 0x31, 0x00, 0x82, 0x16, 0xA1, 0xA1, 0x12
      } }; // Bender's daydream

```

## Handshake

Handshakes are the first messages sent when a peer wants to connect to another peer, as we saw in the [levin](levin.md) chapter a [handshake](levin.md#handshake-1001) request contains the [basic node data](levin.md#basic-node-data) and [core sync data](levin.md#core-sync-data), a response also contains this data but with an extra field, peer list, which is just a list of [peer list entry base](levin.md#peer-list-entry-base)s.

### Initiating a handshake with a peer

when initiating a handshake with a peer first we first get our nodes [basic node data](levin.md#basic-node-data) and [core sync data](levin.md#core-sync-data), this information is then sent to the peer and we wait for a response.

A peer must respond in less than 5 seconds:

```c++
#define P2P_DEFAULT_HANDSHAKE_INVOKE_TIMEOUT            5000       //5 seconds
```

When we receive the response we:
- Check the [basic node data](levin.md#basic-node-data)'s [network_id](#network-id) is correct for the network we are on
- Handle and check the received peers by the function `handle_remote_peerlist` which is explained [here](peer_store.md#handle-remote-peer-list)
- Process the peers core sync data by the function `process_payload_sync_data` which is explained [here](cryptonote_protocol.md#process-core-sync-data)
- We add the peer to our [white peer list](peer_store.md#white-list), and our [anchor peer list](peer_store.md#anchor-list)
- Check If the peer did not send their support flags, if so  we initiate a [support flag request](#support-flags)

### Handling a handshake request

## Support Flags

A support flags request is initiated during a peers handshake for if the peer did not include their support flags in the handshake. A [support flags](levin.md#support-flags-1007) request does not include any information, and a response just contains the support flags, asingle u32. 

### Retrieving A Peers Support Flags

A support flags request requires no data so we just send the levin header with an empty body with the [support flags](levin.md#support-flags-1007) request flag.

This uses the same timeout as a [Handshake request](#initiating-a-handshake-with-a-peer) so the peer must respond in less than 5 seconds:

```c++
#define P2P_DEFAULT_HANDSHAKE_INVOKE_TIMEOUT            5000       //5 seconds
```

When the peer responds we can return the peers support flags to the caller.

### Handling a support flags request

We literally just get out support flags and return it to the peer.