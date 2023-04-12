# Cryptonote Protocol

The Cryptonote protocol contains all the logic for syncing (and staying synced) with peers and passing un-mined transactions around.

## Process Core Sync Data

This is called whenever we receive a peers [Core Sync Data](levin.md#core-sync-data) which is during a [handshake](levin.md#handshake-1001) 
and a [timedsync](levin.md#timed-sync-1002). This function in monerod 
is `process_payload_sync_data` this is what it does:

