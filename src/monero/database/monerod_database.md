## Overview of monerod database

To better understand similarities between Cuprate's database and Monerod one, we first need to take a look at the latter. This chapter is just an overview, for more details on each components being addressed see the corresponding sub-chapter.

***

Monerod (from the core-project) implement its database code under `src/blockchain_db/` folder :

```
src/
└──blockchain_db/
	      └──blockchain_db.h // header of BlockchainDB and DB_EXCEPTION
	      └──blockchain_db.cpp
	      └──lmdb/
			   └──db_lmdb.cpp // Implementation for LMDB
			   └──db_lmdb.h
```

Monerod use **LMDB** (Lightning Memory-Mapped Database), an embedded key-value database with a performance and stability record speaking for itself. It uses a Binary+ Tree to structure its stored data, this make it extremely efficient on disk optimized for random I/O, such as SSD. (*This is why the Monero Community discourage the use of HDD, that is designed for sequential writes. It really put overhead on the disk and dramatically slow down performance and its lifespan*.)

LMDB is lightweight, performant, embedded, ACID, reliable and support Copy-On-Write. So it is natural for the core-project to develop Monerod on top of it. Other options that could have been considered is LevelDB developed by Google, that show better read performance.

### Introduction to Cryptonote Blockchain database interface:

Here's a kind message from the developers that is definitely going to help us understand later subject:

```cpp
/*
 * Cryptonote Blockchain Database Interface
 *
 * The DB interface is a store for the canonical block chain.
 * It serves as a persistent storage for the blockchain.
 *
 * For the sake of efficiency, a concrete implementation may also
 * store some blockchain data outside of the blocks, such as spent
 * transfer key images, unspent transaction outputs, etc.
 *
 * Examples are as follows:
 *
 * Transactions are duplicated so that we don't have to fetch a whole block
 * in order to fetch a transaction from that block.
 *
 * Spent key images are duplicated outside of the blocks so it is quick
 * to verify an output hasn't already been spent
 *
 * Unspent transaction outputs are duplicated to quickly gather random
 * outputs to use for mixins
 *
 * Indices and Identifiers:
 * The word "index" is used ambiguously throughout this code. It is
 * particularly confusing when talking about the output or transaction
 * tables since their indexing can refer to themselves or each other.
 * I have attempted to clarify these usages here:
 *
 * Blocks, transactions, and outputs are all identified by a hash.
 * For storage efficiency, a 64-bit integer ID is used instead of the hash
 * inside the DB. Tables exist to map between hash and ID. A block ID is
 * also referred to as its "height". Transactions and outputs generally are
 * not referred to by ID outside of this module, but the tx ID is returned
 * by tx_exists() and used by get_tx_amount_output_indices(). Like their
 * corresponding hashes, IDs are globally unique.
 *
 * The remaining uses of the word "index" refer to local offsets, and are
 * not globally unique. An "amount output index" N refers to the Nth output
 * of a specific amount. An "output local index" N refers to the Nth output
 * of a specific tx.
 * ...
 */
```

**TL;DR**: For storage efficiency, object identifier used as database key are mapped as 64-bit integer. A Block ID is its height. For optimal performance a lot of data are duplicated into different tables to accelerate gathering by not having to deserialize an entire block if we just want an Output's commitment for example. Spent key images are duplicated to quickly verify if it's been spent. Unspent Outputs are duplicated in a table to quickly gather random ones for mixins.

*If at this state you're lost, you might want to go check the Monero documentation. Speificly the Blockchain part*

### BlockchainDB Interface

Any database implementation in monerod is based on the C++ class **BlockchainDB** (see `blockchain_db/blockchain_db.h|.cpp`). Any database engine can be used in Monerod as long as it implement the private member of BlockchainDB, giving to the rest of the node a variety of useful functions.
Here's the description from monerod of BlockchainDB:

```cpp
/**
 * @brief The BlockchainDB backing store interface declaration/contract
 *
 * This class provides a uniform interface for using BlockchainDB to store
 * a blockchain.  Any implementation of this class will also implement all
 * functions exposed here, so one can use this class without knowing what
 * implementation is being used.  Refer to each pure virtual function's
 * documentation here when implementing a BlockchainDB subclass.
 *
 * A subclass which encounters an issue should report that issue by throwing
 * a DB_EXCEPTION which adequately conveys the issue.
 */
class BlockchainDB
```
BlockchainDB take for unified error interface DB_EXCEPTION, another C++ class that inherit the standard C++ error class:
```cpp
/**
 * @brief A base class for BlockchainDB exceptions
 */
 class DB_EXCEPTION : public std::exception
```
**See [Errors](errors.md) for more details about DB_EXCEPTION**

And this is it. This two definitions are the only one needed for implementing another database engine for monerod. This abstraction has been made to simplify database call in the rest of the node but also add modularity. How the data are stored is up to the specific implementation. Fortunately for us their is only one to explain.

### LMDB Implementation

LMDB code can be found under the `blockchain_db/lmdb/db_lmdb.h|.cpp` files.

The first and most important matter of any key-value database is its database schema, or tables. As of now, we're at the 5th version the database:
```cpp
// Increase when the DB structure changes
#define VERSION 5
```
You can find a comment on it in `db_lmdb.cpp`:
```cpp
/* DB schema:
 *
 * Table            Key          Data
 * -----            ---          ----
 * blocks           block ID     block blob
 * block_heights    block hash   block height
 * block_info       block ID     {block metadata}
 *
 * txs_pruned       txn ID       pruned txn blob
 * txs_prunable     txn ID       prunable txn blob
 * txs_prunable_hash txn ID      prunable txn hash
 * txs_prunable_tip txn ID       height
 * tx_indices       txn hash     {txn ID, metadata}
 * tx_outputs       txn ID       [txn amount output indices]
 *
 * output_txs       output ID    {txn hash, local index}
 * output_amounts   amount       [{amount output index, metadata}...]
 *
 * spent_keys       input hash   -
 *
 * txpool_meta      txn hash     txn metadata
 * txpool_blob      txn hash     txn blob
 *
 * alt_blocks       block hash   {block data, block blob}
 *
 * Note: where the data items are of uniform size, DUPFIXED tables have
 * been used to save space. In most of these cases, a dummy "zerokval"
 * key is used when accessing the table; the Key listed above will be
 * attached as a prefix on the Data to serve as the DUPSORT key.
 * (DUPFIXED saves 8 bytes per record.)
 *
 * The output_amounts table doesn't use a dummy key, but uses DUPSORT.
 */
```
At first glance you might think that this is detailed and complete (and if you perfectly understand congratulation, you're a core dev), but if you, like me, started discovering the node by its database, you'll quickly ask yourself dumb questions such as : 

*block ID is an height, why not  just put "block height" in the Key column then ?*, *Block blob means it is block bytes, this means there is other data that aren't encoded, its not.* *Ok {...} means it is a struct, but which struct?* *what is a dummy key?* *Why "{block metadata}" but then "txn metadata",txpool don't put metadata in a struct?* *[...] means its an array or a tuple ????* *Can I have an explanation of each table purpose?*

Anyway, you might not see it before looking right under this comment, but there is 3 more tables : 
```cpp
const char* const LMDB_BLOCKS = "blocks";
const char* const LMDB_BLOCK_HEIGHTS = "block_heights";
const char* const LMDB_BLOCK_INFO = "block_info";

const char* const LMDB_TXS = "txs";
const char* const LMDB_TXS_PRUNED = "txs_pruned";
const char* const LMDB_TXS_PRUNABLE = "txs_prunable";
const char* const LMDB_TXS_PRUNABLE_HASH = "txs_prunable_hash";
const char* const LMDB_TXS_PRUNABLE_TIP = "txs_prunable_tip";
const char* const LMDB_TX_INDICES = "tx_indices";
const char* const LMDB_TX_OUTPUTS = "tx_outputs";

const char* const LMDB_OUTPUT_TXS = "output_txs";
const char* const LMDB_OUTPUT_AMOUNTS = "output_amounts";
const char* const LMDB_SPENT_KEYS = "spent_keys";

const char* const LMDB_TXPOOL_META = "txpool_meta";
const char* const LMDB_TXPOOL_BLOB = "txpool_blob";

const char* const LMDB_ALT_BLOCKS = "alt_blocks";
/* vvvvvv       here        vvvvvvv */
const char* const LMDB_HF_STARTING_HEIGHTS = "hf_starting_heights";
const char* const LMDB_HF_VERSIONS = "hf_versions";

const char* const LMDB_PROPERTIES = "properties";
```

And if you don't know what is a "zerokval" dummy key, here's an explanation: 

Normally, key-value database permit the access of a value through its key. But imagine if we want multiple values for the same key, how do we procede ? Here's come the cursors, cursors are capable of iterating over duplicated data placed under the same key. A very cool
feature of LMDB is that you can get a data not just by iterating them until finding the good one, but give the prefix of the data you attend to receive. If you do that LMDB return the first (or nearest) duplicated value that have these first bytes you give him earlier. So an optimization to gain space, is to use a *dummy key*. For monerod it is 8 null bytes placed as the only key in the table. Every data will be placed under this dummy key. When you want to get a specific data, you just have to place the cursor under this key and give him the prefix of this data, which, in practice, act as the *real key*. 

**See [Tables](tables.md) subchapter for details of all tables**

These tables implement specific data structures for storing the blockchain.

**See [Types](types.md) subchapter for details of database types**

All private members of BlockchainDB in `db_lmdb.cpp` use a number of macros for clarity

**See [Macros](macros.md) subchapter for macros explanation**