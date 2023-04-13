## Database structures and types

Tables in monerod store data as data structures or types. Some are defined in `blockchain_db.h` other in `db_lmdb.h` and `db_lmdb.cpp`. This chapter try to bring details on each data structures and purpose.
 
Notes: 
- `#pragma pack()` is used to gain place in memory. See [this StackOverflow answer](http://stackoverflow.com/questions/3318410/ddg#3318475) for explanation.
- [cumulative difficulty has been set as a 128bit integer](https://github.com/monero-project/monero/pull/5239), so difficulty_low means the first lower 64bit, difficulty_high means the upper 64bit.


### blockchain_db.h
***

The following types are used in BlockchainDB interface and therefore it's method prototype.

#### relay_category

```cpp
enum class relay_category : uint8_t
{
  broadcasted = 0,//!< Public txes received via block/fluff
  relayable,      //!< Every tx not marked `relay_method::none`
  legacy,         //!< `relay_category::broadcasted` + `relay_method::none` for rpc relay requests or historical reasons
  all             //!< Everything in the db
};
```
An enum used to keep track of where this transaction come from. It is used in different txpool functions. These infos are then stored in
a `txpool_tx_meta_t` using the `void txpool_tx_meta_t::set_relay_method(relay_method method) noexcept {}` method.

#### txpool_tx_meta_t

```cpp
struct txpool_tx_meta_t
{
  crypto::hash max_used_block_id;
  crypto::hash last_failed_id;
  uint64_t weight;
  uint64_t fee;
  uint64_t max_used_block_height;
  uint64_t last_failed_height;
  uint64_t receive_time;
  uint64_t last_relayed_time; //!< If received over i2p/tor, randomized forward time. If Dandelion++stem, randomized embargo time. Otherwise, last relayed timestamp
  // 112 bytes
  uint8_t kept_by_block;
  uint8_t relayed;
  uint8_t do_not_relay;
  uint8_t double_spend_seen: 1;
  uint8_t pruned: 1;
  uint8_t is_local: 1;
  uint8_t dandelionpp_stem : 1;
  uint8_t is_forwarding: 1;
  uint8_t bf_padding: 3;

  uint8_t padding[76]; // till 192 bytes

  /* skip methods */
}
```
A large data structure used to keep metadata of incoming transaction in the Transaction Pool. It is then stored in the `txpool_meta` table.

#### tx_data_t

```cpp
#pragma pack(push, 1)
struct tx_data_t
{
  uint64_t tx_id;
  uint64_t unlock_time;
  uint64_t block_id;
};
```
A data structure used in the `txindex` struct for the `tx_indices` table, to get a transaction ID, its unlock time and origin block.

#### alt_block_data_t

```cpp
struct alt_block_data_t
{
  uint64_t height;
  uint64_t cumulative_weight;
  uint64_t cumulative_difficulty_low;
  uint64_t cumulative_difficulty_high;
  uint64_t already_generated_coins;
};
```
A data structure used in the `alt_block` table to keep track of important metadata for alternative blocks. Essentialy what is mandatory to
decide wether this block lead to reorganize the mainchain or not.

#### output_data_t

```cpp
#pragma pack(push, 1)

/**
 * @brief a struct containing output metadata
 */
struct output_data_t
{
  crypto::public_key pubkey;       //!< the output's public key (for spend verification)
  uint64_t           unlock_time;  //!< the output's unlock time (or height)
  uint64_t           height;       //!< the height of the block which created the output
  rct::key           commitment;   //!< the output's amount commitment (for spend verification)
};
#pragma pack(pop)
```
Output's primary data. used in the `output_amounts` table.

### db_lmdb.cpp
***

The following types and data structures are only used in LMDB implementation

#### mdb_block_info_*

<details>
<summary>Older types</summary>

```cpp
typedef struct mdb_block_info_1
{
  uint64_t bi_height;
  uint64_t bi_timestamp;
  uint64_t bi_coins;
  uint64_t bi_weight; // a size_t really but we need 32-bit compat
  uint64_t bi_diff;
  crypto::hash bi_hash;
} mdb_block_info_1;

typedef struct mdb_block_info_2
{
  uint64_t bi_height;
  uint64_t bi_timestamp;
  uint64_t bi_coins;
  uint64_t bi_weight; // a size_t really but we need 32-bit compat
  uint64_t bi_diff;
  crypto::hash bi_hash;
  uint64_t bi_cum_rct;
} mdb_block_info_2;

typedef struct mdb_block_info_3
{
  uint64_t bi_height;
  uint64_t bi_timestamp;
  uint64_t bi_coins;
  uint64_t bi_weight; // a size_t really but we need 32-bit compat
  uint64_t bi_diff;
  crypto::hash bi_hash;
  uint64_t bi_cum_rct;
  uint64_t bi_long_term_block_weight;
} mdb_block_info_3;
```

</details>

```cpp
typedef struct mdb_block_info_4
{
  uint64_t bi_height; // subkey | block ID
  uint64_t bi_timestamp;
  uint64_t bi_coins;
  uint64_t bi_weight; // a size_t really but we need 32-bit compat
  uint64_t bi_diff_lo;
  uint64_t bi_diff_hi;
  crypto::hash bi_hash;
  uint64_t bi_cum_rct;
  uint64_t bi_long_term_block_weight;
} mdb_block_info_4;

typedef mdb_block_info_4 mdb_block_info;
```
This data structure store block's metadata. Used in `block_info` table. This is the 4th version but the older types are still defined.

#### pre_rct_outkey & pre_rct_output_data_t

```cpp
#pragma pack(push, 1)
// This MUST be identical to output_data_t, without the extra rct data at the end
struct pre_rct_output_data_t
{
  crypto::public_key pubkey;       //!< the output's public key (for spend verification)
  uint64_t           unlock_time;  //!< the output's unlock time (or height)
  uint64_t           height;       //!< the height of the block which created the output
};

typedef struct pre_rct_outkey {
    uint64_t amount_index; // subkey
    uint64_t output_id;
    pre_rct_output_data_t data;
} pre_rct_outkey;
```
These structs is used to store Pre-RingCT output index, and underlying data. see the `output_amount` table in Tables chapter.

#### outkey

```cpp
typedef struct outkey {
    uint64_t amount_index; // subkey
    uint64_t output_id;
    output_data_t data;
} outkey;
```
Exactly the same as `pre_rct_outkey` except, it contains an `output_data_t` which is add the RingCT Key commitment.

#### outtx

```cpp
typedef struct outtx {
    uint64_t output_id; // subkey
    crypto::hash tx_hash;
    uint64_t local_index;
} outtx;
```
Used in `output_txs` table. Just a tuple of output's transaction hash and its local index.

### db_lmdb.h
***

#### txindex

```cpp
typedef struct txindex {
    crypto::hash key; // subkey
    tx_data_t data;
} txindex;
```
The struct used  in `txs_indices`