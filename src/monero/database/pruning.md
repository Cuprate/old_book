# Pruning

Monero pruning works by having 8 posible pruning seed, the seed chosen will decide what part of the blockchains sigining data your node will keep. Each pruned peer generates their pruning seed randomly.

## Stripes 

This is the amount of different blockchain portions that a pruned peer could keep. For Monero this is currently 8 that means the blockchain's signing data is split into 8 portions.

## Stripes Size

This is the amount of continuous blocks in each stripe. For Monero this is currently 4096 that means the first pruning stripe will have blocks 0-4095 etc and prune every block not in it's stripe as shown in the table below:

| stripe           | 1             | 2            | 3            | 4  | 5  | 6  | 7  | 8  |
|------------------|---------------|--------------|--------------|----|----|----|----|----|
| will have blocks | 0 - 4095      | 4095 - 8,190 | 8190 - 12285 | .. | .. | .. | .. | .. |
|                  | 32768 - 36863 | ..           | ..           | .. | .. | .. | .. | .. |
|                  | ..            | ..           | ..           | .. | .. | .. | .. | .. |

## Tip Blocks

Blocks within 5500 of the tip of the chain will not be pruned.

## Generating Pruning Seeds

The function in Monero to generate pruning seeds:

```c++
uint32_t make_pruning_seed(uint32_t stripe, uint32_t log_stripes)
{
  CHECK_AND_ASSERT_THROW_MES(log_stripes <= PRUNING_SEED_LOG_STRIPES_MASK, "log_stripes out of range");
  CHECK_AND_ASSERT_THROW_MES(stripe > 0 && stripe <= (1ul << log_stripes), "stripe out of range");
  return (log_stripes << PRUNING_SEED_LOG_STRIPES_SHIFT) | ((stripe - 1) << PRUNING_SEED_STRIPE_SHIFT);
}
```
This function takes in a stripe which is number 1 to 8 including(1 & 8) and a log_stripes which is log2 of the amount of different stripes (8) which is 3.

The constants used in this function:
```c++
static constexpr uint32_t PRUNING_SEED_LOG_STRIPES_SHIFT = 7;
static constexpr uint32_t PRUNING_SEED_LOG_STRIPES_MASK = 0x7;
static constexpr uint32_t PRUNING_SEED_STRIPE_SHIFT = 0;
```
The possible inputs/ outputs of this function (`log_stripes` is always 3)

| input (stripe) | output (seed) |
|----------------|---------------|
| 1              | 384           |
| 2              | 385           |
| 3              | 386           |
| 4              | 387           |
| 5              | 388           |
| 6              | 389           |
| 7              | 390           |
| 8              | 391           |

## Getting A Seeds Log Stripes

Monero currently only accepts a log stripes of 3 and will reject any peers which have a different log stripes. The function to calculate a seeds log stripes is:

```c++
constexpr inline uint32_t get_pruning_log_stripes(uint32_t pruning_seed) { 
    return (pruning_seed >> PRUNING_SEED_LOG_STRIPES_SHIFT) & PRUNING_SEED_LOG_STRIPES_MASK; 
}

```

This only return 3 for all currently valid monero seeds.

## Getting A Seeds Pruning Stripe

The seeds pruning stripe corresponds to the blocks we keep. This is the function in Monero:
```c++
  inline uint32_t get_pruning_stripe(uint32_t pruning_seed) { 
    if (pruning_seed == 0) return 0; 
    return 1 + ((pruning_seed >> PRUNING_SEED_STRIPE_SHIFT) & PRUNING_SEED_STRIPE_MASK); }
```

A pruning seed of 0 means no pruning. This function is just the inverse of [Generating Pruning Seeds](#generating-pruning-seeds) so the inputs/ outputs of this will just be the other way round. 

## Geting A Blocks Pruning Stripe

The function in Monero to get the pruning stripe that corresponds to keeping that block is:

```c++
uint32_t get_pruning_stripe(uint64_t block_height, uint64_t blockchain_height, uint32_t log_stripes)
{
  if (block_height + CRYPTONOTE_PRUNING_TIP_BLOCKS >= blockchain_height)
    return 0;
  return ((block_height / CRYPTONOTE_PRUNING_STRIPE_SIZE) & (uint64_t)((1ul << log_stripes) - 1)) + 1;
}
```
[Pruning Stripe Size](#stripes-size)

As you can see this function checks if the `block_height` is within [Tip Blocks](#tip-blocks) and returns 0 if it is, this means that every seed will have this block. 

`((1ul << log_stripes) - 1)` This sets the last 3 bits: `0000 0111` so when we bitand we 
remove every other bit.

`(block_height / CRYPTONOTE_PRUNING_STRIPE_SIZE)` For block any block 0 to 4095 this will output `0000 0000` and for any blocks 32768 to 36863 this will output `0000 1000`.

As you can see when we bitand `0000 0000` & `0000 00111` == `0000 1000` & `0000 0111` we can then +1 to get the stripe that will keep those blocks (1). 

## Getting A Blocks Pruning Seed

The Blocks pruning seed is the seed that will keep that block. This is the function in Monero:

```c++
uint32_t get_pruning_seed(uint64_t block_height, uint64_t blockchain_height, uint32_t log_stripes)
{
  const uint32_t stripe = get_pruning_stripe(block_height, blockchain_height, log_stripes);
  if (stripe == 0)
    return 0;
  return make_pruning_seed(stripe, log_stripes);
}
```
This is simple, a call to [`get_pruning_stripe`](#geting-a-blocks-pruning-stripe) and passing that stripe into [`make_pruning_seed`](#generating-pruning-seeds)

# TODO

explain these monsters:

```c++
uint64_t get_next_unpruned_block_height(uint64_t block_height, uint64_t blockchain_height, uint32_t pruning_seed)
{
  CHECK_AND_ASSERT_MES(block_height <= CRYPTONOTE_MAX_BLOCK_NUMBER+1, block_height, "block_height too large");
  CHECK_AND_ASSERT_MES(blockchain_height <= CRYPTONOTE_MAX_BLOCK_NUMBER+1, block_height, "blockchain_height too large");
  const uint32_t stripe = get_pruning_stripe(pruning_seed);
  if (stripe == 0)
    return block_height;
  if (block_height + CRYPTONOTE_PRUNING_TIP_BLOCKS >= blockchain_height)
    return block_height;
  const uint32_t seed_log_stripes = get_pruning_log_stripes(pruning_seed);
  const uint64_t log_stripes = seed_log_stripes ? seed_log_stripes : CRYPTONOTE_PRUNING_LOG_STRIPES;
  const uint64_t mask = (1ul << log_stripes) - 1;
  const uint32_t block_pruning_stripe = ((block_height / CRYPTONOTE_PRUNING_STRIPE_SIZE) & mask) + 1;
  if (block_pruning_stripe == stripe)
    return block_height;
  const uint64_t cycles = ((block_height / CRYPTONOTE_PRUNING_STRIPE_SIZE) >> log_stripes);
  const uint64_t cycle_start = cycles + ((stripe > block_pruning_stripe) ? 0 : 1);
  const uint64_t h = cycle_start * (CRYPTONOTE_PRUNING_STRIPE_SIZE << log_stripes) + (stripe - 1) * CRYPTONOTE_PRUNING_STRIPE_SIZE;
  if (h + CRYPTONOTE_PRUNING_TIP_BLOCKS > blockchain_height)
    return blockchain_height < CRYPTONOTE_PRUNING_TIP_BLOCKS ? 0 : blockchain_height - CRYPTONOTE_PRUNING_TIP_BLOCKS;
  CHECK_AND_ASSERT_MES(h >= block_height, block_height, "h < block_height, unexpected");
  return h;
}

uint64_t get_next_pruned_block_height(uint64_t block_height, uint64_t blockchain_height, uint32_t pruning_seed)
{
  const uint32_t stripe = get_pruning_stripe(pruning_seed);
  if (stripe == 0)
    return blockchain_height;
  if (block_height + CRYPTONOTE_PRUNING_TIP_BLOCKS >= blockchain_height)
    return blockchain_height;
  const uint32_t seed_log_stripes = get_pruning_log_stripes(pruning_seed);
  const uint64_t log_stripes = seed_log_stripes ? seed_log_stripes : CRYPTONOTE_PRUNING_LOG_STRIPES;
  const uint64_t mask = (1ul << log_stripes) - 1;
  const uint32_t block_pruning_seed = ((block_height / CRYPTONOTE_PRUNING_STRIPE_SIZE) & mask) + 1;
  if (block_pruning_seed != stripe)
    return block_height;
  const uint32_t next_stripe = 1 + (block_pruning_seed & mask);
  return get_next_unpruned_block_height(block_height, blockchain_height, tools::make_pruning_seed(next_stripe, log_stripes));
}
```