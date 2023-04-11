# Peer Store

Monero keeps a list of know peers in it's peer store so we can connect to the network without having to connect to seed nodes every time. The Peer Store can be split into 3 seperate groups:

### White List 

The white list contains peers we have connected to in the past, this list is capped in monerod at 1000 entries:

```c++
#define P2P_LOCAL_WHITE_PEERLIST_LIMIT                  1000
```

When connecting to peers monerod trys to make around 70% of our connections from the White Peer List: 

```c++
#define P2P_DEFAULT_WHITELIST_CONNECTIONS_PERCENT       70
```

### Gray List 

The gray list contains peers we have been told about but haven't connected to ourselfs, this list is capped at 5000 entries:

```c++
#define P2P_LOCAL_GRAY_PEERLIST_LIMIT                   5000
```

### Anchor List

The anchor list is the list of previous connections, so when we shutdown and start backup again we can connect to some of the peers we were previously connected to, this helps prevent the node from being isolated by an attacker. When a connection [Handshake](admin_protocol.md#handshake) is completed the peer is appended to this list and when a connection is dropped the peer is dropped from the list. 

monerod tries to maintain at least 2 anchor connections:

```c++
#define P2P_DEFAULT_ANCHOR_CONNECTIONS_COUNT            2
```

## Gray List House Keeping 

Periodically monerod will [ping](admin_protocol.md#ping) gray list peers to check if they are online, if they are then they are added to the white peer list, if they are not then they are dropped from our gray list. This process helps to keep our gray list clear of junk peers.

## Handle Remote Peer List 

This is used when a node receives a peer list from a connected peer.
The node will check that every recived peer is from the correct "zone" (Public|Tor|I2p) if one isn't we ignore the whole peer list and drop the peer that sent the list. 