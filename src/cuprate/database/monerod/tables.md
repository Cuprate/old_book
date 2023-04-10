## LMDB Tables

Monerod use simple tables for storing blob. But metadata are often stored on tables with dummy keys. Such tables are declared with DUPSORT & DUPFIXED flags. DUPSORT is a flag for the database to support duplicated data, DUPFIXED is used when the key is of a fixed size, to gain place
(8 bytes per key).

When talking about Subkey in this chapter, you must understand that the table use a dummy key and that the Subkey used to retrieve data is the first bytes, or prefix, of this data monerod is trying to get.
```
Normal table:
Key -> Value
Duplicate Table:
Dummykey -> {Subkey, Value}
The subkey is the first bytes of the data it tried to store.
```

Here's an explanation of all used flags from the [lmdb crate documentation](https://docs.rs/lmdb/latest/lmdb/struct.DatabaseFlags.html):

**MDB_INTEGERKEY** : Keys are binary integers in native byte order. Setting this option requires all keys to be the same size, typically 32 or 64 bits.

**MDB_DUPFIXED**: This flag may only be used in combination with DUP_SORT. This option tells the library that the data items for this database are all the same size, which allows further optimizations in storage and retrieval. When all data items are the same size, the GET_MULTIPLE and NEXT_MULTIPLE cursor operations may be used to retrieve multiple items at once.

**MDB_DUPSORT**: Duplicate keys may be used in the database. (Or, from another perspective, keys may have multiple data items, stored in sorted order.) By default keys must be unique and may have only a single data item.</br>

## List of tables

### blocks
***
**Flags**: MDB_INTEGERKEY</br>
**Key**: Block's height as `uint_64t` (8 bytes)</br>
**Value**: Block bytes as `cryptonote::Blobdata` (Dynamic size).</br>

This table store the entire blocks. They can be accessed through their height

### blocks_heights
***
**Flags**: MDB_INTEGERKEY | MDB_DUPSORT | MDB_DUPFIXED</br>
**Subkey**: Block's Keccak-256 Hash (32 bytes)</br>
**Value**: Block's height as a 64bit unsigned integer, `uint_64t` (8 bytes).</br>

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
**Subkey**: Block's height as `uint_64t` (8 bytes)</br>
**Value**: Block's metadata as `mdb_block_info_4` (96 bytes).</br>

This table store block's metadata. See types subchapter for more details about it.

todo()!

### txs
***
**Subkey**: Block's Keccak-256 Hash (32 bytes)</br>
**Value**: Block's height as a 64bit unsigned integer, `uint_64t` (8 bytes).</br>