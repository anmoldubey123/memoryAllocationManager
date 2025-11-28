# Memory Pool Manager

## Custom Memory Allocator with First-Fit Strategy

## Overview

A fixed-size memory pool allocator implemented in C that provides `malloc`/`free`-like functionality within a pre-allocated 4KB memory region. The allocator uses a first-fit allocation strategy with block splitting and coalescing to minimize fragmentation.

## Author

- **Anmol Dubey**

## Features

| Feature | Description |
|---------|-------------|
| Fixed Pool Size | 4096-byte pre-allocated memory region |
| First-Fit Allocation | Finds first available block that satisfies request |
| Block Splitting | Divides large blocks to reduce internal fragmentation |
| Block Coalescing | Merges adjacent free blocks to reduce external fragmentation |
| 8-Byte Alignment | All allocations aligned to 8-byte boundaries |
| Script-Driven Testing | Command file interface for automated testing |
| Debug Visualization | Detailed block-by-block memory inspection |
| Fragmentation Metrics | Statistical analysis of pool utilization |

## How It Works

The allocator maintains a doubly-linked list of memory blocks, each tracked by a separate metadata structure. When an allocation request arrives, the allocator searches for the first free block large enough to satisfy the request, optionally splitting it if excess space remains.

```
Initial State (4096 bytes free):
┌────────────────────────────────────────────────────────┐
│                    FREE (4096 bytes)                    │
└────────────────────────────────────────────────────────┘

After MALLOC a 100, MALLOC b 200:
┌──────────┬────────────┬────────────────────────────────┐
│ a (104)  │  b (200)   │         FREE (3792)            │
│ ALLOC    │  ALLOC     │                                │
└──────────┴────────────┴────────────────────────────────┘

After FREE a:
┌──────────┬────────────┬────────────────────────────────┐
│ a (104)  │  b (200)   │         FREE (3792)            │
│ FREE     │  ALLOC     │                                │
└──────────┴────────────┴────────────────────────────────┘

After FREE b (coalescing occurs):
┌────────────────────────────────────────────────────────┐
│                    FREE (4096 bytes)                    │
└────────────────────────────────────────────────────────┘
```

## Data Structures

### MemoryBlock

Metadata structure tracking each block in the pool:

```c
typedef struct MemoryBlock {
    size_t size;              // Payload size (aligned)
    int    is_free;           // 1 = free, 0 = allocated
    struct MemoryBlock* next; // Next block in address order
    struct MemoryBlock* prev; // Previous block in address order
    void* addr_in_pool;       // Pointer to actual memory region
} MemoryBlock;
```

### MemoryPool

Global pool state:

```c
typedef struct {
    void*  pool_start;        // Base address of pool
    size_t total_size;        // Total pool capacity (4096)
    size_t free_size;         // Sum of free block sizes
    MemoryBlock* all_list;    // Head of block list
    int    total_blocks;      // Free + allocated count
    int    free_blocks;       // Free block count
} MemoryPool;
```

## Building and Running

### Compilation

```bash
gcc -Wall -Wextra -g -o manager manager.c
```

### Execution

```bash
./manager
Enter the name of the commands file:
test_commands.txt
```

## Command Reference

The allocator reads commands from a script file:

| Command | Syntax | Description |
|---------|--------|-------------|
| INIT | `INIT` | Initialize the memory pool |
| MALLOC | `MALLOC <id> <size>` | Allocate `size` bytes, store pointer as `id` |
| FREE | `FREE <id>` | Free allocation associated with `id` |
| STATS | `STATS` | Print pool statistics |
| DEBUG | `DEBUG` | Print detailed block list |
| COALESCE | `COALESCE` | Manually trigger block coalescing |
| CLEANUP | `CLEANUP` | Release all pool resources |
| QUIT | `QUIT` | Exit the script |

Lines starting with `#` are treated as comments.

## Example Script

```bash
# test_basic.txt - Basic allocation test
INIT
STATS

MALLOC a 100
MALLOC b 200
MALLOC c 50
STATS
DEBUG

FREE b
STATS
DEBUG

FREE a
FREE c
STATS

CLEANUP
QUIT
```

### Expected Output

```
=== Memory Pool Statistics ===
Total pool size : 4096 bytes
Free memory     : 4096 bytes
Total blocks    : 1
Free blocks     : 1 (scanned=1)
Largest free    : 4096 bytes
Fragmentation   : 0.0%
==============================

MALLOC a 100
MALLOC b 200
MALLOC c 50

=== Memory Pool Statistics ===
Total pool size : 4096 bytes
Free memory     : 3736 bytes
Total blocks    : 4
Free blocks     : 1 (scanned=1)
Largest free    : 3736 bytes
Fragmentation   : 0.0%
==============================

FREE b

=== Memory Pool Statistics ===
Total pool size : 4096 bytes
Free memory     : 3936 bytes
Total blocks    : 4
Free blocks     : 2 (scanned=2)
Largest free    : 3736 bytes
Fragmentation   : 5.1%
==============================
```

## API Functions

### Core Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `pool_init` | `int pool_init(void)` | Initialize pool, returns 0 on success |
| `pool_malloc` | `void* pool_malloc(size_t size)` | Allocate memory, returns pointer or NULL |
| `pool_free` | `void pool_free(void* ptr)` | Free previously allocated memory |
| `pool_cleanup` | `void pool_cleanup(void)` | Release all pool resources |

### Utility Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `pool_stats` | `void pool_stats(void)` | Print utilization statistics |
| `pool_debug` | `void pool_debug(void)` | Print detailed block information |
| `coalesce_blocks` | `void coalesce_blocks(void)` | Merge adjacent free blocks |

## Implementation Details

### Alignment

All allocations are rounded up to 8-byte boundaries:

```c
#define ALIGNMENT 8
#define ALIGN(sz) (((sz) + (ALIGNMENT-1)) & ~(ALIGNMENT-1))
```

A request for 100 bytes becomes 104 bytes after alignment.

### First-Fit Allocation

The allocator traverses the block list sequentially, returning the first free block with sufficient size. This provides O(n) allocation time where n is the number of blocks.

### Block Splitting

When a free block is larger than needed, the excess space becomes a new free block if it meets the minimum size threshold (8 bytes):

```c
void split_block(MemoryBlock* block, size_t size) {
    size_t leftover = block->size - size;
    if (leftover >= ALIGNMENT) {
        // Create new free block from excess
    }
}
```

### Coalescing

After each `pool_free`, adjacent free blocks are merged to combat external fragmentation:

```c
void coalesce_blocks(void) {
    // Traverse list, merge consecutive free blocks
    // that are physically adjacent in memory
}
```

### Error Handling

The allocator detects and reports:
- Double-free attempts
- Freeing invalid pointers
- Out-of-memory conditions

## Fragmentation Calculation

Fragmentation is computed as:

```
fragmentation = 1 - (largest_free_block / total_free_memory)
```

A fragmentation of 0% means all free memory is contiguous. Higher values indicate scattered free blocks that may prevent large allocations despite sufficient total free space.

## Limitations

| Limitation | Details |
|------------|---------|
| Fixed pool size | 4096 bytes, compile-time constant |
| No thread safety | Single-threaded use only |
| Metadata overhead | Each block requires separate 40-byte structure |
| First-fit only | No best-fit or worst-fit alternatives |
| No realloc | Must free and malloc to resize |

## File Structure

```
MemoryPoolManager/
├── README.md           # This file
├── manager.c           # Complete implementation
└── test_commands.txt   # Example test script
```

## Algorithm Complexity

| Operation | Time Complexity | Notes |
|-----------|-----------------|-------|
| pool_init | O(1) | Single allocation |
| pool_malloc | O(n) | First-fit search |
| pool_free | O(n) | Pointer lookup + coalesce |
| coalesce_blocks | O(n) | Single pass through list |
| pool_cleanup | O(n) | Free all metadata |

Where n = number of blocks in the pool.

## Testing Scenarios

### Basic Allocation
```
INIT
MALLOC a 100
MALLOC b 200
FREE a
FREE b
CLEANUP
```

### Fragmentation Test
```
INIT
MALLOC a 100
MALLOC b 100
MALLOC c 100
FREE b          # Creates hole
STATS           # Shows fragmentation
MALLOC d 50     # Fits in hole
CLEANUP
```

### Coalescing Test
```
INIT
MALLOC a 100
MALLOC b 100
MALLOC c 100
FREE a
FREE c
DEBUG           # Two separate free blocks
FREE b
DEBUG           # Single coalesced block
CLEANUP
```

## References

- Kernighan & Ritchie, *The C Programming Language*
- Bryant & O'Hallaron, *Computer Systems: A Programmer's Perspective* (Chapter 9: Virtual Memory)
- Knuth, *The Art of Computer Programming, Vol. 1* (Section 2.5: Dynamic Storage Allocation)


---

*Implementation demonstrates core memory management concepts including allocation strategies, fragmentation management, and metadata tracking.*
