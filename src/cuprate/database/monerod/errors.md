Monerod database implement an error template called `DB_EXCEPTION` (see `blockchain_db.h`). This class permit the node to identify errors from the database and react acordingly to these events. This chapter is just to bring some details on each errors:

The **DB_EXCEPTION** Definition : 
```cpp
/**
 * @brief A base class for BlockchainDB exceptions
 */
class DB_EXCEPTION : public std::exception
{
  private:
    std::string m;

  protected:
    DB_EXCEPTION(const char *s) : m(s) { }

  public:
    virtual ~DB_EXCEPTION() { }

    const char* what() const throw()
    {
      return m.c_str();
    }
};
```

### DB_ERROR

```cpp
/**
 * @brief A generic BlockchainDB exception
 */
class DB_ERROR : public DB_EXCEPTION
{
  public:
    DB_ERROR() : DB_EXCEPTION("Generic DB Error") { }
    DB_ERROR(const char* s) : DB_EXCEPTION(s) { }
};
```
This error is returned whenever lmdb failed to do an operation, such as opening a table, put, get. You'll likely find it in the following format:
`DB_ERROR(lmdb_error("message from the dev explaining what operation failed"))`

### DB_ERROR_TXN_START

```cpp
/**
 * @brief thrown when there is an error starting a DB transaction
 */
class DB_ERROR_TXN_START : public DB_EXCEPTION
{
  public:
    DB_ERROR_TXN_START() : DB_EXCEPTION("DB Error in starting txn") { }
    DB_ERROR_TXN_START(const char* s) : DB_EXCEPTION(s) { }
};
```
Only used in `batch_abort()`,`block_rtxn_start()`,`block_wtxn_start()`,`block_wtxn_stop()`,`block_wtxn_abort()` and `add_block`. 

### DB_OPEN_FAILURE

```cpp
/**
 * @brief thrown when opening the BlockchainDB fails
 */
class DB_OPEN_FAILURE : public DB_EXCEPTION
{
  public:
    DB_OPEN_FAILURE() : DB_EXCEPTION("Failed to open the db") { }
    DB_OPEN_FAILURE(const char* s) : DB_EXCEPTION(s) { }
};
```
This error is returned throw when monerod isn't able to create the database directory (insufficient permissions), when the database is already opened by another process (lmdb lock) or when it is corrupted.
```cpp
...
throw0(cryptonote::DB_OPEN_FAILURE((lmdb_error(error_string + " : ", res) + std::string(" - you may want to start with --db-salvage")).c_str()));
...
throw0(DB_OPEN_FAILURE(std::string("Failed to create directory ").append(filename).c_str()));
...
throw0(DB_OPEN_FAILURE("Attempted to open db, but it's already open"));
```

### DB_CREATE_FAILURE

```cpp
/**
 * @brief thrown when creating the BlockchainDB fails
 */
class DB_CREATE_FAILURE : public DB_EXCEPTION
{
  public:
    DB_CREATE_FAILURE() : DB_EXCEPTION("Failed to create the db") { }
    DB_CREATE_FAILURE(const char* s) : DB_EXCEPTION(s) { }
};
```
Not in use.

### DB_SYNC_FAILURE

```cpp
/**
 * @brief thrown when synchronizing the BlockchainDB to disk fails
 */
class DB_SYNC_FAILURE : public DB_EXCEPTION
{
  public:
    DB_SYNC_FAILURE() : DB_EXCEPTION("Failed to sync the db") { }
    DB_SYNC_FAILURE(const char* s) : DB_EXCEPTION(s) { }
};
```
Not in use.

### BLOCK_DNE

```cpp
/**
 * @brief thrown when a requested block does not exist
 */
class BLOCK_DNE : public DB_EXCEPTION
{
  public:
    BLOCK_DNE() : DB_EXCEPTION("The block requested does not exist") { }
    BLOCK_DNE(const char* s) : DB_EXCEPTION(s) { }
};
```
Used everywhere a block wasn't found.

### BLOCK_PARENT_DNE

```cpp
/**
 * @brief thrown when a block's parent does not exist (and it needed to)
 */
class BLOCK_PARENT_DNE : public DB_EXCEPTION
{
  public:
    BLOCK_PARENT_DNE() : DB_EXCEPTION("The parent of the block does not exist") { }
    BLOCK_PARENT_DNE(const char* s) : DB_EXCEPTION(s) { }
};
```
Only used one time in `add_block()` function when the specified parent of the block is the top block (*grandfather paradox* it's impossible)
```cpp
    blk_height *prev = (blk_height *)parent_key.mv_data;
    if (prev->bh_height != m_height - 1)
      throw0(BLOCK_PARENT_DNE("Top block is not new block's parent"));
```

### BLOCK_EXISTS

```cpp
/**
 * @brief thrown when a block exists, but shouldn't, namely when adding a block
 */
class BLOCK_EXISTS : public DB_EXCEPTION
{
  public:
    BLOCK_EXISTS() : DB_EXCEPTION("The block to be added already exists!") { }
    BLOCK_EXISTS(const char* s) : DB_EXCEPTION(s) { }
};
```
Only used one time `add_block` function when the specified block is already present in the database.
```cpp 
throw1(BLOCK_EXISTS("Attempting to add block that's already in the db"));
```

### BLOCK_INVALID

```cpp
/**
 * @brief thrown when something is wrong with the block to be added
 */
class BLOCK_INVALID : public DB_EXCEPTION
{
  public:
    BLOCK_INVALID() : DB_EXCEPTION("The block to be added did not pass validation!") { }
    BLOCK_INVALID(const char* s) : DB_EXCEPTION(s) { }
};
```
Not in use.

### TX_DNE

```cpp
/**
 * @brief thrown when a requested transaction does not exist
 */
class TX_DNE : public DB_EXCEPTION
{
  public:
    TX_DNE() : DB_EXCEPTION("The transaction requested does not exist") { }
    TX_DNE(const char* s) : DB_EXCEPTION(s) { }
};
```
Only used in `remove_transaction_data()`,`get_tx_unlock_time()` and `get_tx_block_height()`.

### TX_EXISTS

```cpp
/**
 * @brief thrown when a transaction exists, but shouldn't, namely when adding a block
 */
class TX_EXISTS : public DB_EXCEPTION
{
  public:
    TX_EXISTS() : DB_EXCEPTION("The transaction to be added already exists!") { }
    TX_EXISTS(const char* s) : DB_EXCEPTION(s) { }
};
```
Only used in `add_transaction_data()` function when a item can be fetch with the same transaction ID:

```cpp
result = mdb_cursor_get(m_cur_tx_indices, (MDB_val *)&zerokval, &val_h, MDB_GET_BOTH);
  if (result == 0) {
    txindex *tip = (txindex *)val_h.mv_data;
    throw1(TX_EXISTS(std::string("Attempting to add transaction that's already in the db (tx id ").append(boost::lexical_cast<std::string>(tip->data.tx_id)).append(")").c_str()));
```

### OUTPUT_DNE

```cpp
/**
 * @brief thrown when a requested output does not exist
 */
class OUTPUT_DNE : public DB_EXCEPTION
{
  public:
    OUTPUT_DNE() : DB_EXCEPTION("The output requested does not exist!") { }
    OUTPUT_DNE(const char* s) : DB_EXCEPTION(s) { }
};
```
This is error is returned when an output has been found in the database with the corresponding key and subkey (amount and amount idx). see Tables chapter for more details.

### OUTPUT_EXISTS

```cpp
/**
 * @brief thrown when an output exists, but shouldn't, namely when adding a block
 */
class OUTPUT_EXISTS : public DB_EXCEPTION
{
  public:
    OUTPUT_EXISTS() : DB_EXCEPTION("The output to be added already exists!") { }
    OUTPUT_EXISTS(const char* s) : DB_EXCEPTION(s) { }
};
```
Not in use.

### KEY_IMAGE_EXISTS

```cpp
/**
 * @brief thrown when a spent key image exists, but shouldn't, namely when adding a block
 */
class KEY_IMAGE_EXISTS : public DB_EXCEPTION
{
  public:
    KEY_IMAGE_EXISTS() : DB_EXCEPTION("The spent key image to be added already exists!") { }
    KEY_IMAGE_EXISTS(const char* s) : DB_EXCEPTION(s) { }
};
```
Only used in `add_spent_key()` function when the key image already exist. It can't, unless the universe gave us a collision, but that is a legend only cryptographer dreamed of :
```cpp
  MDB_val k = {sizeof(k_image), (void *)&k_image};
  if (auto result = mdb_cursor_put(m_cur_spent_keys, (MDB_val *)&zerokval, &k, MDB_NODUPDATA)) {
    if (result == MDB_KEYEXIST)
      throw1(KEY_IMAGE_EXISTS("Attempting to add spent key image that's already in the db"));
```


