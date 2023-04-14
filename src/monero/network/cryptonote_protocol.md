# Cryptonote Protocol

The Cryptonote protocol contains all the logic for syncing (and staying synced) with peers and passing un-mined transactions around.

## Connection State

This is an enum for what is currently happing between ours and the peers node:

```c++
enum state
{
    state_before_handshake = 0, //default state
    state_synchronizing,
    state_standby,
    state_idle,
    state_normal
};
```

## Peer Score 

Monero uses a scoring system to allow some light mistakes but if a peer makes too many of these mistakes we drop them. 

Currently the amount of light mistakes a peer can make is 2:

```c++
#define DROP_PEERS_ON_SCORE -2
```

## Process Core Sync Data

This is called whenever we receive a peers [Core Sync Data](levin.md#core-sync-data) which is during a [handshake](levin.md#handshake-1001) 
and a [timedsync](levin.md#timed-sync-1002). This function in monerod 
is called `process_payload_sync_data` this is what it does:

`process_payload_sync_data` returns a bool and if that bool is false the peer is dropped.

- First we check if the peer is in [state](#connection-state): `state_before_handshake`, if the peer is we return true. 
*note this function takes in a value `is_initial` for if the peer is currently doing a handshake so we can ignore this check* 
- We then check if the peer is in [state](#connection-state): `state_synchronizing`, if we are we return true.
- If the peers advertised top-version is greater than 6 we then check if the top-version is what we expect it to be for it's height, if it isn't we return false. 
- If the peer has pruning enabled, we check the peers pruning seed is standard:
```c++
const uint32_t log_stripes = tools::get_pruning_log_stripes(hshd.pruning_seed);
if (log_stripes != CRYPTONOTE_PRUNING_LOG_STRIPES || tools::get_pruning_stripe(hshd.pruning_seed) > (1u << log_stripes))
{
MWARNING(context << " peer claim unexpected pruning seed " << epee::string_tools::to_string_hex(hshd.pruning_seed) << ", disconnecting");
return false;
}
```

We check the amount of stripes the peers seed is using is 8, by getting the log stripes of the seed and seeing if it equals 3. We then check if the peers stripe is invalid by seeing if it's greater than the total amount of stripes (8).

> link to pruning chapter

- We then check if the peers height has dropped by comparing it to our stored peer height, if it has dropped we drop the [peers score](#peer-score) by 1. 
- We then set the stored peers height and pruning seed
- Next we get our target block height, if the target is 0 we set it to our block height:
```c++
uint64_t target = m_core.get_target_blockchain_height();
if (target == 0)
    target = m_core.get_current_blockchain_height();
```
- We then check if we have the peers top block, if we do we set the peers [state](#connection-state) as `state_normal` and return true.
- If the network zone is anything other than `public_` we set the peers [state](#connection-state) as `state_normal` and return true, as no chain synchronization is done over Tor/ I2p.
- Next we see if the peers height is greater than `target` if it is:
    - If `is_initial` We print a message at `Info` level if not we use `Debug` that a new top block candidate was found
    - If we are further than 5 blocks away we set safesyncmode to false.
    - If `m_core.get_target_blockchain_height() == 0` we initialize the syncing variables:
        ```c++
        if (m_core.get_target_blockchain_height() == 0) // only when sync starts
        {
        m_sync_timer.resume();
        m_sync_timer.reset();
        m_add_timer.pause();
        m_add_timer.reset();
        m_last_add_end_time = 0;
        m_sync_spans_downloaded = 0;
        m_sync_old_spans_downloaded = 0;
        m_sync_bad_spans_downloaded = 0;
        m_sync_download_chain_size = 0;
        m_sync_download_objects_size = 0;
        }
        ```
    - We set our target height to the peers height.
- If we have set the node to not sync we set the [state](#connection-state) at `state_normal` and return true.
- We then set the [state](#connection-state) to `state_synchronizing`
- We then send a callback request to `p2p_endpoint_t`.