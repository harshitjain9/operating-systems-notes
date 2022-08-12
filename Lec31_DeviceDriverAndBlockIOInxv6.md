**File systems and I/O in xv6**
- Multiple layers of abstraction in file systems
    * System call implementations (open, read, write)
    * Operations on file system data structures (inodes, files, directories)
    * Block I/O layer (in-memory cache of disk blocks)
    * Device driver (communicates with hard disk to read/write blocks)
- This lecture and next: overview of these various layers in xv6 file system

**Disk blocks and buffers**
- Disk maintains data as 512-byte blocks
- Disk buffer (struct buf) = <u>copy of disk block in memory</u>
- Buffer cache (bcache) is an <u>array of disk buffers</u>
    - Pointers across buffers create a linked list, most recently used buffers at head
- <u>Reading a block from disk</u>: assign buffer for the block number in buffer cache, device driver sends request to disk controller, disk controller raises interrupt when data is ready, data copied from disk controller into buffer cache (VALID flag set after data is read- meaning this entry in the cache has valid data)
- <u>Writing a block to disk</u>: first write into buffer in buffer cache, device driver copies data from buffer to disk controller, disk controller raises interrupt when write completes (DIRTY flag is set until disk is updated)

 **Device driver**
 - Process that wishes to read/write calls iderw function, buffer as argument
    * If dirty flag is set, write request. If valid flag not set, read request
    * Requests added to queue, function idestart issues requests one after another
    * Process sleeps until request completes
- Within idestart, <u>communication with disk controller registers via in/out instructions</u>

**Device driver (continued)**
- When disk controller completes read/write operation, it raises an interrupt
    * Data is read from disk controller into buffer using "in" instruction
    * Process sleeping for data is woken up
    * Next request from queue is issued
- No support for DMA in x86. With DMA, <u>data is copied by disk controller into memory buffers directly before raising interrupt</u>
    * Interrupt handler need not copy data using I/O instructions

**Disk buffer cache: block read/write**
- <u>All processes access disk via buffer cache only</u>
- <u>Only copy of disk block in cache, only one process can access it at a time</u>
    * Different processes do not have their separate copies of disk block
- Process calls "bread" to read a disk block, which calls function "bget"
    * Function bget returns buffer if it already exists in cache and no other process using it
    * If valid buffer not returned by bget, read from disk
- Process calls "bwrite" to write a block to disk: set dirty bit and request device driver to write
- When done with block, process calls brelse to release block so that some other process can access this disk block, moves to head of list (most recently used)


**Disk buffer cache: block read/write- bget in more detail**
- Function bget returns pointer to a disk block if it exists in the cache
    * Uses a bcache.lock to ensures only one process at a time accesses a disk buffer
- If block in cache and another process using it, sleep until the block is released by the other process
- If block not in cache, find a least recently used non-dirty buffer and recycle it to use for this block
- Two goals achieved by buffer cache
    * <u>Recently used disk blocks are stored in memory for future use</u>
    * <u>Disk blocks modified by one process at a time</u>

**Logging layer (overview)**
- A system call can change multiple blocks at a time on disk, and we want atomicity in case the system crashes during a system call. Either all changes are made or none is made. 
    * Example: we do not want disk block added to the inode of a file but the file data not yet written to it
- Logging ensures atomicity by grouping disk block changes into transactions
    * Every system call starts a transaction in the log, writes all changed disk blocks in the log, and commits the transaction
    * Later, the log installs the changes in the original disk blocks one by one
    * If crash happens before log is written fully, no changes made
    * If crash happens after log entry is committed, log entries are replayed when system restarts after crash
- In xv6, changes of multiple system calls are collected in memory and committed to log together. Actual changes happen to disk blocks only after the group transaction commits
    * Process must call "log_write" instead of "bwrite" during system call

**Summary**
- Device driver in xv6 communicates with disk controller using in/out instructions to read/write disk blocks
    * Simple driver with no DMA capability
- Buffer cache stores all recently read disk blocks in memory, and synchronizes access to disk blocks across processes
- All blocks changed in a system call are logged on disk and changes are installed atomically
- Next: <u>file system code translates system calls into block read/write operations</u>