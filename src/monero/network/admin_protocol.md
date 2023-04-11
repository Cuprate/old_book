# Admin Protocol

The Monero admin protocol contains the logic for connecting to peers and making sure peers are staying active.

## Network ID

Monero has IDs for it's different networks which are sent in handshakes to make sure we are connecting to peers on the smae network.
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

Handshakes are the first messages that are sent when a peer wants to connect to another peer, as we saw in the [levin](levin.md) chapter a [handshake](levin.md#handshake-1001) request contains the [basic node data](levin.md#basic-node-data) and [core sync data](levin.md#core-sync-data), a response also contains this data but with an extra field, peer list, which is just a list of [peer list entrie base](levin.md#peer-list-entry-base)s.

### Initiating a handshake with a peer

when initiating a handshake with a peer first we pust get our nodes [basic node data](levin.md#basic-node-data) and [core sync data](levin.md#core-sync-data) this infomation is then sent to the peer and we wait for a response.

A peer must respond in less than 5 seconds:

```c++
#define P2P_DEFAULT_HANDSHAKE_INVOKE_TIMEOUT            5000       //5 seconds
```

When we receive the response we:
- Check the [basic node data](levin.md#basic-node-data)'s [network_id](#network-id) is correct for the network we are on
- Handle and check the received peers by the function `handle_remote_peerlist` which is explained [here]()
- Process the peers core sync data by the function `process_payload_sync_data` which is explained [here]()
- We add the peer to our [white peer list](peer_store.md#white-list)
- Check If the peer did not send their support flags, if so  we initiate a [support flag request]()

