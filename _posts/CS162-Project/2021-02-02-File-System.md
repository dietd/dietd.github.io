---
title: "File System"
layout: page
---

Task 1: Buffer Cache
====================================


**1. Data structures and functions:**
In cache.c:
```C
struct cache_block {
struct lock lock_cache_block;         /* Lock the cache block to ensure coherency of the blocks. */
uint8_t block_data[BLOCK_SECTOR_SIZE];    /* Stores the data */     
        bool recently_used;           /* Stores if the data was recently used */
        block_sector_t sector_index;    /* Stores the index of the sector in disk */
        bool valid, dirty;                  /* Bookkeeping */
}
size_t clock_head;                          /* Used to keep track of who to evict. */
void cache_init(void) {
* Initialize our lock_cache lock.
* Iterate through our cache 
* Setting each block as invalid
* Initialize the cache blocks lock.
}
struct cache_block caches[64];      /* Defines the whole cache, can be used  */
struct lock lock_cache;                  /* Locks the full cache to ensure that threads do not attempt to evict the same block twice */ 
size_t evict_cache_block(block_sector_t index) {
* See the Clock algorithm in our algorithms section
}
```
typedef uint32_t block_sector_t;


**2. Algorithms:**
* In file_sys.c change the init function to call cache_init().
* Also change the inode read and write functions to check our cache using the evict algorithm before reading from the disk.
* Clock algorithm:
* Set all recently_used for all pages to be 1.
* After a fault in finding the sector (the block is not there or the block is invalid), advance the clock hand until we find a page that we can replace with the new sector
* The clock hand moves from one page to another. If the recently_used for a page is 1 and it is valid, set it to 0 else 
* On eviction, if the cache_block is dirty, write the block to disk.
* Return the index of the cache block we found.
* On shutdown, we need to write back dirty blocks to the disk, which is done in filesys_done().


**3. Synchronization:**
* We have a lock for each of our cache blocks individually for when a thread wants to modify the data stored in them, and we also have a full cache lock for when we evict a block from our cache.


**4. Rationale:**
* Why is it better than the alternative?
        * It is easy to implement but can be improved if we use a hashmap to sort the sectors instead of an array. We can also change this to be nth chance pretty easily by adding a counter that counts down every iteration of the clock.
* Shortcomings?
        * Accessing and modifying the array may be slow.


Task 2: Extensible Files
====================================
## To Implement:


## Inumber Syscall
See Task 3 for implementation.


##To Modify (in inode.c):


We will try to have 124 direct blocks, 1 indirect block, and 1 double indirect block for each inode.


## byte_to_sector()
**1. Data structures and functions:**
```C
struct inode_disk *disk_data;                    /* Stores the data */
struct inode_disk {
        //All block_sector_t add up to 126
        block_sector_t direct[124];           /* 124 because stack align */
        block_sector_t indirect;                 /* 1 also because stack align */
        block_sector_t doubly_indirect;    /* 1 also because stack align */
        off_t length;                                   /* 1 also because stack align (off_t is a u_int32_t so it is 4 bytes) */
        unsigned magic;                       /* magic makes everything better */
}
```
## How this adds up to at least 8MB:
Each indirect block struct points to 128 blocks (making it a full sector of block_sector_t). We have 124 direct blocks, 1 indirect block which points to another struct that has 128 blocks. The doubly indirect block points to another block which points to another one thus 128 * 128. In total we have (124 + 128 + 128 * 128) = 16636 sectors being pointed to. This is equivalent to 8.5176 MB of data that we can store and access.


**2. Algorithms:**
* We define a inode_disk pointer that we malloc to hold the data from the cache.
* Then we read from the cache into the disk_data we malloced.
* Replace inode->data with disk_data which we just read.
* Use the data to calculate the sector and free the disk_data we malloced.
* Return the calculated value. 


 ## How will the math work when converting from bytes to sectors?
Get the position of the inode and divide by sector number. This will be our index into the current inode.
Suppose our index is within the range [0, direct_blocks_size]. That would mean it’s using the direct blocks and we can directly access our sector number by directly indexing into the inode.
For indices that are in the indirect block, which is in indexes [direct_block_size, direct_block_size + indirect_block_size], we need to first read the block at the indirect block sector and then index into it by the indirect index.
For indices that are in the doubly indirect blocks, we have to load two indirect blocks using the cache and then index into both of them.


## inode_open()
**2. Algorithms:**
* Remove the call to block_read(fs_device, inode->sector, &inode->data)


## inode_close()
**1. Data structures and functions:**
```C
struct inode_disk *disk_data;            /* Stores the data */
```


**2. Algorithms:**
* We define a inode_disk pointer that we malloc to hold the data from the cache.
* Then we read from the cache into the disk_data we malloced.
* Replace inode->data with disk_data which we just read.
* Use the data to calculate the sector and return the calculated value after freeing the space we malloced.


## inode_length()
**1. Data structures and functions:**
```C
struct inode_disk *disk_data;            /* Stores the data */
```


**2. Algorithms:**
* We define a inode_disk pointer that we malloc to hold the data from the cache.
* Then we read from the cache into the disk_data we malloced.
* Replace inode->data with disk_data which we just read.
* Store the data->length to a temporary variable and free the space we malloced.
* Return the temporary variable.


## File Growth


We need to modify free_map_allocate().


Basic Idea:
Change free_map_allocate() to our own function that scans for total sectors not being used and writes to the array of sectors inputted. Flip those sectors to false, and then return success if there are as many sectors as we need. 


Functions we will need to change:
## inode_create()
* In inode_create what we want to do is to call our function that populates the inode called allocate_inode().
* Next we want to write the thing to block using the cache. 


## inode_read_at()
* malloc a buffer for an inode, call the cache to read the data into the cache. We would also like to check if the offset is invalid, meaning that the offset is greater than the length of the file.


## inode_write_at()
* If we are writing past the end of the file:
*  We could want to use our function allocate_inode() to allocate an additional block
        *  Call our beautiful function buffer_read() to read the inode into an allocated kernel buffer.
        * Then zero-pad our file based on the amount of space we skipped past the EOF.
        * Then start writing to the newly allocated inode using buffer_write().


This function will try to allocate a sector and return its success
## allocate_sector(block_sector_t * sector_index):
* Scan through the free_map of the sectors on disk, and return a valid sector that can be used. Write this sector index into the array using the cache.


## allocate_inode(size_t cnt, struct inode_disk * inode_disk)
* If any of the calls to allocate_sector() fails we then return NULL and revert to the previous state.
* Save the number of sectors needed based on the length of the file in a temporary variable. 
* Start allocating the direct blocks
        * Iterate through the direct_block list on the disk_inode and call allocate_sector() on each 
Element.
* Start allocating the indirect blocks,
        * Read into the indirect block using the cache. 
* Then allocate the number of sectors needed in that indirect block using allocate_sector. 
* Write the result to disk using the cache.
* Start allocating the double indirect blocks
         * Read into the double indirect block we allocated through cache. 
* Allocate a sector the first indirect block. Then for each block in that indirect block allocate a sector each for those. Then finally for each newly created second level indirect block, we need to fill them using allocate_sector().
* Write the indirect block to cache.


## deallocate_inode(struct inode * inode);
* We loop through each level of the inode structure and use bitmap_reset on each sector of that.


**3. Synchronization:**
* We will have a free_map lock to ensure that the bitmap for the sectors is being changed using one thread at a time.


**4. Rationale:**
* Why is it better than the alternative?
        * There seems to be no alternatives to writing into the blocks. 
*Another approach might just be mallocing a whole sector into memory and then writing our disk inode in that buffer, then sending the whole thing back to cache.
* Shortcomings?
        * Speed depends on the speed of the cache as well as the number of sectors needed. So it might be a bit slower than a normal system depending on the parameters.


•You are already familiar with handling memory exhaustion in C, by checking for aNULLreturnvalue frommalloc. In this project, you will also need to handle disk space exhaustion. When yourfile system is unable to allocate new disk blocks, you must have a strategy to abort the currentoperation and rollback to a previous good state.
We gotta use deallocate_inode whenever something fails.


Task 3: Subdirectories
====================================
## To Implement:


Add to thread.h:
```C
struct thread {
struct dir *working_dir;                     /* Keep track of cwd */
}
```


Add to directory.c:
```C
struct dir {
        struct lock dir_lock;                           /* Lock for synchronization on directories */
}
```
Add to inode.c:
```C
struct inode_disk {
        block_sector_t parent_working_dir;   /* inode_disk sector for parent working directory */
        bool isdir;                                            /* Flag to track if this file is a directory */
}
```


## Helper Functions:


## struct dir * get_directory(char * path)
* If the path has a ‘..’ we need to use the inode_disk parent_working_dir and append the new stuff to the end. 
* If the path has a ‘.’ we need to use the working_dir of the thread.
* Then take the path and split it up by ‘/’ and store the parts into an array, using the given get_next_part() code in the spec.
* Start at the root or the current working directory of the process.
* Iteratively call dir_lookup on the directory and the name of the directory in the array.
        * If we cannot find that directory we will error out and return NULL.
* Once we reach the end of the array, the inode is our wanted one. Create a directory struct for this and return it.


## Chdir Syscall
**1. Data structures and functions:**
```C
bool sys_chdir(const char *name);
```


**2. Algorithms:**
* Set current thread’s working_dir to the result of get_directory() if the return is non-null.


## Mkdir Syscall
**1. Data structures and functions:**
```C
sys_mkdir(const char *name);
// Modify to include is_directory flag
filesys_create(const char *name, off_t initial_size, bool is_directory) 
inode_create(block_sector_t sector, off_t length, bool is_directory)
```


**2. Algorithms:**
* Call filesys_create() and set the isdir flag to true. Filesys_create() will fail if the file exists, so we do not need to check.
* We can mark it as a directory with a third parameter that just acts as a flag.


##  Readdir Syscall
**1. Data structures and functions:**
```C
sys_readdir(const char *name)
```


**2. Algorithms:**
* Call get_directory() to get the directory that we want to read from.
* Then call dir_readdir() in directory.c on that directory we found given that it is non-null.


##  Isdir Syscall
**1. Data structures and functions:**
```C
inode_isdirectory(struct inode inode)
```


**2. Algorithms:**
* Use inode_isdirectory() which simply extracts the is_dir variable from the disk_inode


## To modify:


##  filesys_open Syscall
**2. Algorithms:**
* Call get_directory() on the input and store that in a temporary variable.
* Then call dir_open() on that directory we found if it is not null.
* Then we can call file_open(filename)


**3. Synchronization:**
* Use directory’s dir_lock to ensure the directory (and its contents) aren’t modified as we perform our operations on it






##  filesys_close Syscall
**2. Algorithms:**
* Check if the input is a directory, if it is we return NULL and do nothing. 


##  Exec Syscall
**2. Algorithms:**
* Call get_directory(). 
* Compare the result to the current processes working directory. 
* If it is different save the current directory and chdir into the new one. 
* Then run exec normally, but chdir back after we finish executing.


##  Remove Syscall
**2. Algorithms:**
* Using get_directory(), we check if it is a directory and if it is we must find the places that reference them and set them to NULL.
* Otherwise, if it is a file we just remove it normally.


##  Inumber Syscall
**1. Data structures and functions:**
* inode_get_inumber (const struct inode *inode)
* file_get_inode (struct file *file)


**2. Algorithms:**
* Get the file by calling readdir() with the fd 
* Call file_get_inode(file) to get the inode associated with the file
* Call inode_get_inumber using the inode and return the result 


Additional Questions:
====================================
1. For this project, there are 2 optional buffer cache features that you can implement: write-behind and read-ahead. A buffer cache with write-behind will periodically flush dirty blocks to the filesystem block device, so that if a power outage occurs, the system will not lose as much data. Without write-behind, a write-back cache only needs to write data to disk when (1) the data is dirty and gets evicted from the cache, or (2) the system shuts down. A cache with read-ahead will predict which block the system will need next and fetch it in the background. A read-ahead cache can greatly improve the performance of sequential file reads and other easily-predictable file access patterns. Please discuss a possible implementation strategy for write-behind and a strategy for read-ahead. You must answer this question regardless of whether you actually decide to implement these features.


**For write-behind:**
Write-behind can be implemented by setting a timer on a dispatched thread for a certain amount of kernel ticks. This thread would then call an interrupt which uses the timer interrupt, which then flushes the cache to the disk. This is the simplest implementation. We could also implement it as part of our scheduler and have it run whenever there is nothing else to run along with a timer that counts down from the last time we flushed. We could have our scheduler push the cache to disk whenever there is a gap in stuff to execute in order to maximize our utilization.


**For read-ahead:**
We would try to dispatch a thread that will read 1 block before and after the one we accessed and put that into the cache. So in inode_read, we would want a helper function that asynchronously evicts blocks in the cache and replaces it with the blocks before and after the address of the offset. This would be implementing spatial locality because nearby blocks are more likely to be used in the future. This helper function would consist of a call to byte_to_sector() on the two blocks, which returns the sectors. Then, we load the blocks into the cache, so that it can be used later on when the system progresses through the execution of the current process.
