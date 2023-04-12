Monerod database implement an error template called `DB_EXCEPTION` (see `blockchain_db.h`). This class permit the node to identify errors from the database and react acordingly to these events. This chapter is just an explanation of these errors:


The entire section in `blockchain_db.h`
```cpp
/***********************************
 * Exception Definitions
 ***********************************/

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

/**
 * @brief A generic BlockchainDB exception
 */
class DB_ERROR : public DB_EXCEPTION
{
  public:
    DB_ERROR() : DB_EXCEPTION("Generic DB Error") { }
    DB_ERROR(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when there is an error starting a DB transaction
 */
class DB_ERROR_TXN_START : public DB_EXCEPTION
{
  public:
    DB_ERROR_TXN_START() : DB_EXCEPTION("DB Error in starting txn") { }
    DB_ERROR_TXN_START(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when opening the BlockchainDB fails
 */
class DB_OPEN_FAILURE : public DB_EXCEPTION
{
  public:
    DB_OPEN_FAILURE() : DB_EXCEPTION("Failed to open the db") { }
    DB_OPEN_FAILURE(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when creating the BlockchainDB fails
 */
class DB_CREATE_FAILURE : public DB_EXCEPTION
{
  public:
    DB_CREATE_FAILURE() : DB_EXCEPTION("Failed to create the db") { }
    DB_CREATE_FAILURE(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when synchronizing the BlockchainDB to disk fails
 */
class DB_SYNC_FAILURE : public DB_EXCEPTION
{
  public:
    DB_SYNC_FAILURE() : DB_EXCEPTION("Failed to sync the db") { }
    DB_SYNC_FAILURE(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when a requested block does not exist
 */
class BLOCK_DNE : public DB_EXCEPTION
{
  public:
    BLOCK_DNE() : DB_EXCEPTION("The block requested does not exist") { }
    BLOCK_DNE(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when a block's parent does not exist (and it needed to)
 */
class BLOCK_PARENT_DNE : public DB_EXCEPTION
{
  public:
    BLOCK_PARENT_DNE() : DB_EXCEPTION("The parent of the block does not exist") { }
    BLOCK_PARENT_DNE(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when a block exists, but shouldn't, namely when adding a block
 */
class BLOCK_EXISTS : public DB_EXCEPTION
{
  public:
    BLOCK_EXISTS() : DB_EXCEPTION("The block to be added already exists!") { }
    BLOCK_EXISTS(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when something is wrong with the block to be added
 */
class BLOCK_INVALID : public DB_EXCEPTION
{
  public:
    BLOCK_INVALID() : DB_EXCEPTION("The block to be added did not pass validation!") { }
    BLOCK_INVALID(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when a requested transaction does not exist
 */
class TX_DNE : public DB_EXCEPTION
{
  public:
    TX_DNE() : DB_EXCEPTION("The transaction requested does not exist") { }
    TX_DNE(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when a transaction exists, but shouldn't, namely when adding a block
 */
class TX_EXISTS : public DB_EXCEPTION
{
  public:
    TX_EXISTS() : DB_EXCEPTION("The transaction to be added already exists!") { }
    TX_EXISTS(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when a requested output does not exist
 */
class OUTPUT_DNE : public DB_EXCEPTION
{
  public:
    OUTPUT_DNE() : DB_EXCEPTION("The output requested does not exist!") { }
    OUTPUT_DNE(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when an output exists, but shouldn't, namely when adding a block
 */
class OUTPUT_EXISTS : public DB_EXCEPTION
{
  public:
    OUTPUT_EXISTS() : DB_EXCEPTION("The output to be added already exists!") { }
    OUTPUT_EXISTS(const char* s) : DB_EXCEPTION(s) { }
};

/**
 * @brief thrown when a spent key image exists, but shouldn't, namely when adding a block
 */
class KEY_IMAGE_EXISTS : public DB_EXCEPTION
{
  public:
    KEY_IMAGE_EXISTS() : DB_EXCEPTION("The spent key image to be added already exists!") { }
    KEY_IMAGE_EXISTS(const char* s) : DB_EXCEPTION(s) { }
};

/***********************************
 * End of Exception Definitions
 ***********************************/
```


