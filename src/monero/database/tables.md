## LMDB Tables

> Be aware that the current database schema might change when Seraphis is implemented. See: [https://github.com/seraphis-migration/wallet3/issues/11](https://github.com/seraphis-migration/wallet3/issues/11)

Monerod stores every type of dynamic size in simple key-value tables. But any constant sized items (such as block's metadata for example) are stored as duplicated values in tables using dummy keys. Such tables are declared with DUPSORT & DUPFIXED flags. DUPSORT is a flag for the database to support duplicated data, DUPFIXED is used when the data is of a fixed size, to gain space
(8 bytes per key).

When a table, in this chapter, has a Subkey without a Key that means the table uses the `Dummy Key`. This means that the actual key to the data is the first bytes of the data and the "Key" is the `Dummey Key`.

##### Normal table:

```
Key -> Value
```

##### Duplicate Table:

```
Dummykey -> {Subkey, Value}
The subkey is the first bytes of the value that is stored.
```

Here's an explanation of all used flags from the [lmdb crate documentation](https://docs.rs/lmdb/latest/lmdb/struct.DatabaseFlags.html):

**MDB_INTEGERKEY** : Keys are binary integers in native byte order. Setting this option requires all keys to be the same size, typically 32 or 64 bits.

**MDB_DUPFIXED**: This flag may only be used in combination with DUP_SORT. This option tells the library that the data items for this database are all the same size, which allows further optimizations in storage and retrieval. When all data items are the same size, the GET_MULTIPLE and NEXT_MULTIPLE cursor operations may be used to retrieve multiple items at once.

**MDB_DUPSORT**: Duplicate keys may be used in the database. (Or, from another perspective, keys may have multiple data items, stored in sorted order.) By default keys must be unique and may have only a single data item.</br>

## List of tables

### blocks
***
**Flags**: MDB_INTEGERKEY</br>
**Key**: Block's height as `uint64_t` (8 bytes)</br>
**Value**: Block bytes as `cryptonote::Blobdata` (Dynamic size).</br>

This table store the entire blocks. They can be accessed through their height

### blocks_heights
***
**Flags**: MDB_INTEGERKEY | MDB_DUPSORT | MDB_DUPFIXED</br>
**Subkey**: Block's Keccak-256 Hash as `char [32]` (32 bytes)</br>
**Value**: Block's height as a 64bit unsigned integer, `uint64_t` (8 bytes).</br>

This table define a relation for finding a block height based on its Keccak-256 Hash. The first version was a simple table
but the new one store through Duplicated data :
```cpp
	/* old table was k(hash), v(height).
     * new table is DUPFIXED, k(zeroval), v{hash, height}.
     */
```

### blocks_info
***
**Flags**: MDB_INTEGERKEY | MDB_DUPSORT | MDB_DUPFIXED</br>
**Subkey**: Block's height as `uint64_t` (8 bytes)</br>
**Value**: Block's metadata as `mdb_block_info_4` (96 bytes).</br>

This table store [block's metadata](types.md#mdb_block_info_).

### txs
***
**Flags**: MDB_INTEGERKEY</br>
**Key**: Transaction's ID as `uint64_t` (8 bytes)</br>
**Value**: Transaction's bytes as `cryptonote::Blobdata` (Dynamic size).</br>

This table was originally used to store transactions but has been deprecated in favor of txs_pruned and txs_prunable tables.

### txs_pruned
***
**Flags**: MDB_INTEGERKEY</br>
**Key**: Transaction's ID as `uint64_t` (8 bytes)</br>
**Value**: Transaction's Pruned part as `cryptonote::Blobdata` (Dynamic size).</br>

Used to store Pruned part of transaction.

### txs_prunable
***
**Flags**: MDB_INTEGERKEY</br>
**Key**: Transaction's ID as `uint64_t` (8 bytes)</br>
**Value**: Transaction's prunable part as `cryptonote::Blobdata` (Dynamic size).</br>

Used to store prunable part of transaction (Signatures).

### txs_prunable_hash
***
**Flags**: MDB_INTEGERKEY | MDB_DUPSORT | MDB_DUPFIXED</br>
**Subkey**: Transaction's ID as `uint64_t`(8 bytes)</br>
**Value**: Transaction's prunable part hash as `char [32]` (32 bytes).</br>

Store hash of prunable part of transactions (an hash of the signatures blob).

### txs_prunable_tip
***
**Flags**: MDB_INTEGERKEY | MDB_DUPSORT | MDB_DUPFIXED</br>
**Subkey**: Transaction's ID as `uint64_t` (8 bytes)</br>
**Value**: Blocks height as `uint64_t` (8 bytes).</br>

This table stores the height for a transaction if that transaction is withing a certain amount of non-pruned tip blocks.

[Tip blocks](pruning.md#tip-blocks) are blocks at the top of the chain currently 5500 that won't get pruned. 


### tx_indices
***
**Flags**: MDB_INTEGERKEY | MDB_DUPSORT | MDB_DUPFIXED</br>
**Subkey**: Transaction's Hash as `char [32]`(8 bytes)</br>
**Value**: Transaction's ID unlock time, and block's height as `tx_data_t` (24 bytes).</br>

This table define the relation between transaction's hash (that can be found in blocks) and its transaction ID, transaction's unlock time and origin block's height.</br>
[tx_data_t](types.md#tx_data_t)

### tx_outputs
***
**Flags**: MDB_INTEGERKEY</br>
**Key**: Transaction's ID as `uint64_t` (8 bytes)</br>
**Value**: Transaction's output amount idx as a `vector` of `uint64t` (Dynamic Size)</br>

This table is used when deleting a transaction to find the right outputs to delete.

### output_txs
***
**Flags**: MDB_INTEGERKEY | MDB_DUPSORT | MDB_DUPFIXED</br>
**Subkey**: Output's global index (database-only type) as `uint64_t` (8 bytes)</br>
**Value**: Transaction's hash and output's local index as `(char [32], uint64_t)` (40 bytes).</br>

This table is used to retrieve an ouput's transaction and its corresponding local index in this transaction.

### output_amounts
***
**Flags**: MDB_INTEGERKEY | MDB_DUPSORT | MDB_DUPFIXED</br>
**Key**: Output's amount as `uint64_t` (8 bytes)</br>
**Subkey**: Output's amount index as `uint64_t` (8 bytes)</br>
**Value**: Output's metadata as `output_data_t` (PreRCT: 42 bytes, PostRCT: 80 bytes).</br>

This table store the actual output data. Before RingCT, the outputs was sorted by their amount and then their index for this specific amount. (this is why there is a Key **and** a Subkey). All post-RingCT output have an amount of 0 and are therefore stored under the zero key.

[output_data_t](types.md#output_data_t)</br>
[pre_rctoutput_data_t](types.md#pre_rct_outkey--pre_rct_output_data_t)

### spent_keys
***
**Flags**: MDB_INTEGERKEY | MDB_DUPSORT | MDB_DUPFIXED</br>
**Subkey**: Output's key image</br>
**Value**: None (0 bytes).</br>

This table exist to keep track of spent key image. It is used to quickly check for double spends.

### alt_blocks
***
**Flags**: None</br>
**Key**: Alternative Block's hash as `char [32]` (32 bytes)</br>
**Value**: AltBlock's metadata and blob as `{alt_block_data_t, cryptonote::Blobdata}` (40 bytes + Dynamic size)</br>

Used to store alternative blocks broadcasted in the network, since they might (if the required difficulty is achieved) reorganize the mainchain.</br>
[alt_block_data_t](types.md#alt_block_data_t)

### txpool_meta
***
**Flags**: None</br>
**Key**: Transaction's Keccak-256 hash as `char [32]` (32 bytes)</br>
**Value**: Transaction's metadata as `txpool_meta_t` (192 bytes)</br>

This table store the metadata of transactions currently in the transaction pool.</br>
[txpool_meta_t](types.md#txpool_tx_meta_t)

### txpool_blob
***
**Flags**: None</br>
**Key**: Transaction's Keccak-256 hash as `char [32]` (32 bytes)</br>
**Value**: Transaction's blob as `cryptonote::Blobdata` (Dynamic size)</br>

This table store the bytes of transactions currently in the transaction pool.

### hf_versions
***
**Flags**: MDB_INTEGERKEY</br>
**Key**: Block's height as `uint64_t`(8 bytes)</br>
**Value**: HardFork version of this block as `uint8_t`(1 byte)</br>

This table keep track of hard fork versions of blocks in the blockchain.

### hf_starting_heights
***
**Flags**: None</br>
**Key**: HardFork version as `uint8_t` (1 byte)</br>
**Value**: Starting height of the hardfork as `uint64_t` (8 bytes)</br>

This table was used to store hardfork starting height. But it was deprecated, since these were hardcoded into the node. In fact, if a node get a block with an unknown hardfork version, it automatically discard it.

### properties
***

This table is used to store the pruning seed and database version:</br>
**Key**: *version* -> **Value**: version of the database as `uint32_t`</br>
**Key**: *pruning_seed* -> **Value**: pruning seed as `uint32_t`</br>
Version is defined as follow in `db_lmdb.cpp`:

```cpp
#define VERSION 5
#define MDB_val_str(var, val) MDB_val var = {strlen(val) + 1, (void *)val}

/* ... */

  MDB_val_str(k, "version"); // MDB_val k = {strlen("version") + 1, (void *)"version"}
  MDB_val v;
  auto get_result = mdb_get(txn, m_properties, &k, &v);
```
And the pruning seed key is :
```cpp
MDB_val_str(k, "pruning_seed"); // MDB_val k = {strlen("pruning_seed") + 1, (void *)"pruning_seed"}
```
</br>
</br>
</br>

All flags has been extracted from the `BlockchainLMDB::open()` function in `db_lmdb.cpp`:
```cpp
void BlockchainLMDB::open(const std::string& filename, const int db_flags)
{
  
  ...

  // open necessary databases, and set properties as needed
  // uses macros to avoid having to change things too many places
  // also change blockchain_prune.cpp to match
  lmdb_db_open(txn, LMDB_BLOCKS, MDB_INTEGERKEY | MDB_CREATE, m_blocks, "Failed to open db handle for m_blocks");

  lmdb_db_open(txn, LMDB_BLOCK_INFO, MDB_INTEGERKEY | MDB_CREATE | MDB_DUPSORT | MDB_DUPFIXED, m_block_info, "Failed to open db handle for m_block_info");
  lmdb_db_open(txn, LMDB_BLOCK_HEIGHTS, MDB_INTEGERKEY | MDB_CREATE | MDB_DUPSORT | MDB_DUPFIXED, m_block_heights, "Failed to open db handle for m_block_heights");

  lmdb_db_open(txn, LMDB_TXS, MDB_INTEGERKEY | MDB_CREATE, m_txs, "Failed to open db handle for m_txs");
  lmdb_db_open(txn, LMDB_TXS_PRUNED, MDB_INTEGERKEY | MDB_CREATE, m_txs_pruned, "Failed to open db handle for m_txs_pruned");
  lmdb_db_open(txn, LMDB_TXS_PRUNABLE, MDB_INTEGERKEY | MDB_CREATE, m_txs_prunable, "Failed to open db handle for m_txs_prunable");
  lmdb_db_open(txn, LMDB_TXS_PRUNABLE_HASH, MDB_INTEGERKEY | MDB_DUPSORT | MDB_DUPFIXED | MDB_CREATE, m_txs_prunable_hash, "Failed to open db handle for m_txs_prunable_hash");
  if (!(mdb_flags & MDB_RDONLY))
    lmdb_db_open(txn, LMDB_TXS_PRUNABLE_TIP, MDB_INTEGERKEY | MDB_DUPSORT | MDB_DUPFIXED | MDB_CREATE, m_txs_prunable_tip, "Failed to open db handle for m_txs_prunable_tip");
  lmdb_db_open(txn, LMDB_TX_INDICES, MDB_INTEGERKEY | MDB_CREATE | MDB_DUPSORT | MDB_DUPFIXED, m_tx_indices, "Failed to open db handle for m_tx_indices");
  lmdb_db_open(txn, LMDB_TX_OUTPUTS, MDB_INTEGERKEY | MDB_CREATE, m_tx_outputs, "Failed to open db handle for m_tx_outputs");

  lmdb_db_open(txn, LMDB_OUTPUT_TXS, MDB_INTEGERKEY | MDB_CREATE | MDB_DUPSORT | MDB_DUPFIXED, m_output_txs, "Failed to open db handle for m_output_txs");
  lmdb_db_open(txn, LMDB_OUTPUT_AMOUNTS, MDB_INTEGERKEY | MDB_DUPSORT | MDB_DUPFIXED | MDB_CREATE, m_output_amounts, "Failed to open db handle for m_output_amounts");

  lmdb_db_open(txn, LMDB_SPENT_KEYS, MDB_INTEGERKEY | MDB_CREATE | MDB_DUPSORT | MDB_DUPFIXED, m_spent_keys, "Failed to open db handle for m_spent_keys");

  lmdb_db_open(txn, LMDB_TXPOOL_META, MDB_CREATE, m_txpool_meta, "Failed to open db handle for m_txpool_meta");
  lmdb_db_open(txn, LMDB_TXPOOL_BLOB, MDB_CREATE, m_txpool_blob, "Failed to open db handle for m_txpool_blob");

  lmdb_db_open(txn, LMDB_ALT_BLOCKS, MDB_CREATE, m_alt_blocks, "Failed to open db handle for m_alt_blocks");

  // this subdb is dropped on sight, so it may not be present when we open the DB.
  // Since we use MDB_CREATE, we'll get an exception if we open read-only and it does not exist.
  // So we don't open for read-only, and also not drop below. It is not used elsewhere.
  if (!(mdb_flags & MDB_RDONLY))
    lmdb_db_open(txn, LMDB_HF_STARTING_HEIGHTS, MDB_CREATE, m_hf_starting_heights, "Failed to open db handle for m_hf_starting_heights");

  lmdb_db_open(txn, LMDB_HF_VERSIONS, MDB_INTEGERKEY | MDB_CREATE, m_hf_versions, "Failed to open db handle for m_hf_versions");

  lmdb_db_open(txn, LMDB_PROPERTIES, MDB_CREATE, m_properties, "Failed to open db handle for m_properties");

  ...

}
```