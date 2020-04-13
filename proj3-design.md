# Project 3 Design 
## Task 1: Buffer Cache
### Data Structures and Functions
> Metadata for entry:

```C
int used = 0
int chance = 1
lock evicting
```

### Algorithms
For task 1, we are going to use a clock algorithm in order to support the cache replacement policy. Cache cannot be greater than 64 sectors, so it will be 64 * 512 B = 2^15B. 

> On a `read` or `write` call

* We get the `fd (per-process)` index and access the file struct from list of files,
* Then, we go to the list of open `struct inodes` (per_filesys) from the file struct we got
* We can finally access the data blocks and `inode_disk` struct

For the Clock algorithm:
* We are going to implement a cache of file blocks and keep an array of those blocks. Blocks need to have metadata bit (`dirty = 1, clean = 0`) We will have a global lock for the whole cache, and we will acquire the lock when we are evicting it. 
* We traverse through the list just as we are traversing through the pages in the clock algorithm. We will have an index that keeps track of the clock pointer
```C
WHILE (page not found):
    IF (dirty bit == 0):
        REPLACE current page
    ELSE:
        Dirty bit == 0 (change dirty bit to 0)
    INCREMENT clock_pointer_index
```
### Synchronization
On a write call, we do not directly write to the disk, but we only add to disk when `filesys_done` is called. We are going to add the writing feature in `filesys_done()`.

In order to make our buffer cache periodically flush dirty cache blocks to disk, we will flush the dirty blocks whenever the size of the block `inode_read()` is reading is smaller than that of what is left to read in the block. 
### Rationale
We decided to use a clock algorithm because it’s more efficient than LRU. There is no need to maintain a linked list data structure and maintain invariants for the least recently used page. By using a partition, the clock algorithm avoids the above problem while giving us an accurate enough approximation.

## Task 2: Extensible Files
### Data Structures and Functions

| Start            | End              | Sector           |
| ---------------- | ---------------- | ---------------- |
| 0                | 511              | `unused[0]`      |
| 512              | 1023             | `unused[1]`      |
| ...             | ...             | ...             |
| 62464             | 62975             | `unused[122]`             |
| 62976             | 63487             | `unused[123]->data0->data0`|
| 63488             | 64000             | `unused[123]->data0->data4 `            |
| ...             | ...             | ...             |
| 8451072             | 8451583             | `unused[123]->data512->data508`             |
### Algorithms
> How a file grows:

When a file grows and cannot fit in the original blocks any more
1. Load the file’s inode into memory
1. Allocate data blocks on disk by looking up free blocks and marking those blocks as allocated in the free map
1. Modify inode to point to those blocks
1. Write data to those blocks
1. Flush all changes to disk

If the file is larger than 62975B, we may also need to allocate indirect blocks and double indirect blocks in addition to data blocks 

### Synchronization
* When the operating system runs out of disk space, we assign new blocks on the fly.
* When creating a file, if we cannot find enough blocks in the free map, we simply return fail in `file_create()`
* When growing a file, if we cannot find enough blocks (# available blocks < # total blocks we need) we can write data to available blocks and return the number of bytes we have written in `file_write_at()`. Next time we can pick up from where we left when more blocks are freed.


### Rationale 
Here we are going to modify struct `inode_disk` in `inode.c`. More detail will be discussed in Task 3. For now since we need to support up to 8MB, a double indirect pointer is necessary. Meanwhile, we want to minimize the total IOs during reading, so we decide to make the other 123 pointers direct pointers and put them before the double indirect pointer.  In summary,  we are going to split 124 inode’s pointers to 123 direct inode pointers and one double indirect pointer. This allows us to support up to a single file of up to 8.06MB. The specific offset each pointer points to is shown below.

## Task 3 Subdirectories
### Data Structures and Functions
For task 3 we will add an `int is_dir` into struct `inode_disk`. If `is_dir == 0`, then it is a file; otherwise this inode will be a directory.
We are also going to add `struct inode_disk *cwd` into `thread` structure.

`isdir()` will `return is_dir != 0`;

`chdir()` will change the `current_thread()->cwd` to that inode if it exists.
`mkdir()` will enter through the directory until there are not `/` left, then create an `inode` with:
```C
		is_dir = 1;
		inode.unused[0] = cwd’s inode;
		inode.unused[1] = inode;
```
Then we will add `inode` into cwd’s inode’s next free `unused`. If failed, free all resources and `return -1`. No matter what happens, return to the previous directory.


### Algorithms
`readdir()` will check `isdir()`, than scan through all the unused from the corresponding inode and put the file name into the name; 

> On `open()`:
```C
lock_aquire(evicating)
If (cache is still there)
    cache->used++;
lock_release(evicating)
```

> On `close()`:
```C
lock_aquire(evicating)
cache->used--;
lock_release(evicating)
```

For system call `inumber(int fd)`, we are going to return `inode->data->start` as the inumber, since every inode will have a different start sector. 
We allow deleting `cwd` for running process for` remove()`, so we are just going to call `dir_remove()` on `remove()` if `is_dir != 0`. If it is a file, we are going to check if the file is opened and then call `dir_remove()`.
On `exec()`, we are going to call` dir_add()`, and set `is_dir = 0`.

### Synchronization

**We are going to keep the cache synchronized by using a lock for every entry and one global lock for the cache.**

### Rationale 
In task 3, we decided to add a `is_dir` into inode as a way to identify files and directory because it is easy to do, and we will only lose 512 bytes of upper limit for the file size. We added metadata for the cache inorder to keep the synchronization. We can do that because cache will only be in memory, so each cache entry can be larger than 512 bytes. We decided to use more space for the metadata instead of ordering them into in used half and not insured, because that is really hard to implement. The disadvantage of our current solution is that we will lose 1 to 2 entries as they are turned into metadata, but since the loss is so small, it won’t affect the performance as much.

## Additional Questions:
1. Write-behind implementation: timer interrupt every 10s for dirty blocks to be written back to disk. Assign a lock (binary semaphore) to each block: the interrupt has to acquire the lock that belongs to the block before it is being written back to disk. By doing so, the user program would be prevented from writing to the block that is being written back to disk.
1. Read-ahead implementation: implement pre-fetching in`pintos/src/devices/ide.c`. Specifically, create a list of pointers to the blocks that will be pre-fetched. Call `inode_read()` to populate the list whenever there is free space in the current block; i.e. when the size of the block `inode_read()` is reading is smaller than that of what is left to read in the block. Finally, in the interrupt handler we check the status of the list of pointers and periodically load the blocks that they are pointing into the cache. By doing so, we have pre-fetched the blocks that will be read in the future into the buffer. 























