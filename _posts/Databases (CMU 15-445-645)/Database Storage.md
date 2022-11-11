#### Dealing with the limits of hardware

![[Database Storage 2022-11-10 10.46.08.excalidraw]]

A Disk based architecture assumes that the primary storage location of the database is on non-volatile disk
This requires the system to coordinate the movement of data between disk and main memory
	 Note: mmap bad -> we don't want to rely on the OS to determine which memory should be flushed to disk, the database system has a much better understanding of a more optimal policy
The cognitive and latency overhead associated with this managing this is significant (and why some still opt for mmap)
 - From a cognitive perspective it requires careful and explicit    bookkeeping of data
	 - Retaining lookup tables in memory (maps to location on disk)
	 - Determining when to flush data back out to disk
	 - Limiting the amount of data outstanding in memory in individual components, as to not limit other parts of the system
 - Disk based operations (especially on spinning disk) will incur a significant i/o penalty
	 - We want to leverage the benefits of sequential access to speed up reads/writes
	 - We want to do as many operations on data that exists in volatile (main) memory as possible

#### Buffer Pools

![[Database Storage 2022-11-10 11.00.25.excalidraw]]

List of pages stored in main memory that stores the file pages as they are brought into memory for operations, as well as all the metadata needed to track these pages.
	Note: This should sound a lot like the concept of Virtual Memory in the OS, but the database is maintaining this on it's own because it has knowledge that will allow it to manage this memory more efficiently than the OS. Realistically, mmap would be good enough for read only work loads, but performance is significantly worse when we have multiple writers

#### Aside: mmap i/o problems
1. Transaction Safety
	- OS can flush dirty pages at any time, this could allow it to flush dirty pages to disk before a transaction is committed. This could result in some pages for a partially applied transaction to be persisted, while others are still stuck in memory. When the system crashes in this state, the atomic nature of a transaction is violated. This is known as torn writes.
2. I/O Stalls
	- DBMS doesn't know which pages are in memory. When pages are fetched that are not in memory (page fault), the OS will stall the thread as it brings the page in from disk. This can lead to significant reduction in performance of queries.
3. Error Handling
	- Pages are difficult to validate, any access (through the OS) can cause a SIGBUS that the DBMS must handle throughout the code base. When implemented within the database, this check must only be done in the Disk Manager portion of the code base.
		 Note: Bus Error (SIGBUS) - occur when a process is trying to access memory that the CPU cannot physically address. In other words the memory tried to access by the program is not a valid memory address. It caused by alignment issues with the CPU.
1. Performance Issues
	1. OS data structure contention
		1. Want to avoid sharing memory mapping structures (and their eviction policies) with other processes in the OS, as they may conflict with optimal database operation
	2. TLB shoot-downs
		1. **Translation Lookaside Buffer** - memory cache that stores the recent translations of virtual memory to physical memory (1 per core)
		2. **Shootdown** - when a process/core changes the mapping, it must invalidate the cache entry (so that other threads/cores do not attempt to access invalid pages on disk)

---
## File System

DBMS stores a database as one or more files on disk (typically in a proprietary format). The OS is completely ignorant of the contents of these files
- Sqlite stores the entire DB in a single file
- Postgres, MySQL and others typically use multiple files
- Some early systems (1980s) used completely custom file systems on raw storage

### Storage Manager
- Responsible for maintaining the db's files
- Reads and writes can also be scheduled here to improve spatial and temporal locality of the pages
- Organizes files as a collection of pages
	- Tracks active reads & writes from concurrent threads and available space within pages

### Pages
- Fixed sized block of data
- Can contain anything (tuples, meta-data, indexes, log records)
- Typically does not mix different data types within a single page
- Some systems require a page to be self contained
	- This means that the page itself contains all the information needed for it to be correctly interpreted.
	- Requires that the page hold metadata about the data type, data layout within the page, etc.
- Each page is given a unique id
	- An indirection layer is used to male page ids to physical locations

##### 3 Notions of a "Page"
1. Hardware page (usually 4KB)
	-  Smallest amount of data that the OS guarantees it can write out atomically
2. OS Page (also usually 4KB, meant to correlate with HW page)
3. Database Page (512B-16KB)
	- Larger page sizes:
		- allow for fewer system calls to write out data
		- fewer entries in page tables for a fixed amount of data
		- sizes that exceed that of a HW page are no longer atomic
		- the database must implement additional logic to reinsure atomicity

##### Page Storage Architecture
1. Heap File Organization
	- Tuples are unsorted, data can be laid out in any manner within a page
	- Unordered collection of pages with tuples that are stored in random order
	- Basic operations - Create/Get/Write/Delete Page
	- Must also support iterating over all pages - to support sequential scans
	- For a single file DB -> Offset = Page # x Page Size
	- For multiple files we need to track additional metadata to find the requisite page
	- Page Directory
		- Special pages that track the location of data pages in the database files
		- Must make sure that the directory pages are in sync with the data pages
		- Also records metadata about available space (# of free slots in a page, free pages)
1. Tree File Organization
2. Sequential/Sorted File Organization
3. Hashing File Organization
At this level the manager doesn't need to be aware of what is inside the pages

#### Page Structure

![[Database Storage 2022-11-10 17.09.13.excalidraw]]

##### Header
- Page Size
- Checksum
- DBMS Version
- Transaction Visibility (for MVCC)
- Compressions Info
- Some DBs require these pages to be self contained -> allows for disaster recovery in extreme cases
	- Database identifier
	- Table identifier
	- Table Schema
	- Column semantics

##### Tuple Oriented Layout
Slotted Pages
- Supports variable length tuples
- Slot array at the beginning of the page (following the header) 
- Each slot
	- maps to the tuples' starting position offset within the page
	- stores the length of the tuple that it maps to
- Header must also track
	- Number of used slots
	- The offset of the starting location of the last used slot
- Slot array grows towards bottom of page, tuples grow towards the top
- Eventually you run out of space in the middle of the page

Record IDs
- The database needs a way to keep track of individual tuples
- Most common: page_id + offset/slot
- Can also contain information about file location
- Applications should not rely on any meaning in these ids. The system are free to change these at any time they want

Tuple Layout
- A tuple is just a sequence of bytes
- Header (metadata)
	- Visibility info (Concurrency Control)
	- Bit map for null values
	- Do not need info about the schema (should be stored at a higher level)
	- Pointer to overflow data (very large tuples)
	- Order of fields generally respect the order they are specified
		- Might be more efficient to store them differently (but simplicity is usually the priority)
- Denormalized tuple data
	- Can store related tuples together in the same page
	- Can reduce the amount of I/O for a common workload (queries that often/always request both tuples)
	- Can make updates more expensive -> have to fanout writes
	- This storage structure will be hidden from the higher layers of the database system (it will still appear to be logically separate tables)

Inserting a tuple
1. Check page directory to find a page with a free slot
2. Retrieve the page from the disk (if it is not already in the buffer pool)
3. Check the slot array to find an empty space in a page that will fit
Updating a tuple
1. Check page directory to find location of page
2. Retrieve the page from disk (if not already in buffer pool)
3. Find the offset in the page using the slot array
4. Overwrite existing data (assuming the new variable length tuple fits)
	1. If the tuple does not fit in the space at the previous offset, find another space in the page and delete the old one
	2. If it does not fit in the page, find a new page to insert it

Problems with slotted page design
- Fragmentation
	- single tuple may be spread across multiple pages because it exceeds size restrictions of a page
- Useless Disk I/O
	- must bring in the entire page to update a single tuple in the page
- Random disk I/O
	- random layout could lead to updating a page per tuple 



##### Log Oriented Layout

What if the DBMS could not overwrite data in pages, but could only create new pages?

Only two allowed operations PUT & DELETE
- Each log record must contain the tuples unique id
- PUT records contain the tuples contents
- DELETE simply mark the tuple as deleted
- Log entries are appended to the end of the file without any checks on previous entries in the log
- When the page gets full, the DBMS flushes it out to disk and begins writing to a new page
- On disk pages are immutable
- individual pages are (generally) sorted in temporal order

This is significantly more efficient for writes:
- Operate on a single page that is in main memory
- No need for lookups in a page directory
- In memory page is flushed to sequential offset on disk

Reads now require scanning the log to find the requisite data
- Naive: Scan from newest to oldest
- Better: Maintain an index that maps a tuple id to the newest log record
	- If log record is in-memory, just read it
	- If log record is on a disk page, retrieve it
	- Should also persist the index to avoid rescanning the entire log history to reconstruct on restart

Compaction
- Under this architecture, the log will grow forever
- The DBMS needs to reclaim space by consolidating logs that reference the same record id
- Retain only the most recent log entry for a single tuple when compacting pages
	- Can remove delete entries if it is certain that no entry in a previous page exists
- Once a page has undergone some compaction, each record id is guaranteed to only appear once, and therefore the page is not strictly required to be ordered temporally from thereon out
	- **Sorted String Tables**
	- It may be beneficial to order the page by id to improve future lookups
		- Scans within a page are safe to exit early (and move to the next page) if an id is seen that is greater than the record id of the query
- Universal Compaction
	- Compaction can happen between any two page files that are adjacent in time
	- Output is a single sorted run whose time range spreads its inputs
- Level Compaction
	- Compaction occurs dependent on log file sizes
	- Smallest files are compacted at level 0, can compact output with another file at level 1

![[Database Storage 2022-11-11 10.11.33.excalidraw]]

Downsides
- Write Amplification
	- Overhead for compaction of records that are never changed
	- Must be read into memory and written back out to disk even though no compaction on the record occurred
	- Amplification - single insert can result in writing the record as many times as it takes part in compaction
- Compaction is expensive

##### Data Representation

INTEGER/BIGINT/SMALLINT/TINYINT
- c/c++ representation
FLOAT/REAL vs NUMERIC/DECIMAL
- IEEE-754 Standard / Fixed-point Decimals
VARCHAR/VARBINARY/TEXT/BLOB
- Header with length followed by data bytes
- Ned to worry about collations/sorting
TIME/DATE/TIMESTAMP
- 32/64 bit integer of (micro) seconds since UNIX epoch

Large Values (Most systems do not allow a tuple to exceed the size of a single page)
- Overflow Pages
	- the original tuple stores a pointer in the value to the location in the overflow page
	- can link overflow pages together for values that exceed the size of an overflow page
- External Value Storage
	- Some systems allow really large values to be stored in files that are not managed by the dbms
	- Treated as a blob file
	- Does not maintain transactional and durability guarantees of pages/files managed by the dbms

System Catalogs
The dBMS stores metadata about databases in its internal catalogs
- tables, columns, indexes, views
- users & permissions
- internal statistics
Most databases store this info inside itself (as metadata tables -> INFORMATION_SCHEMA)
- Wrap object abstraction around tuples
- Specialized code for "bootstrapping" catalog tables 
	- chicken before the egg, how to interact with these tables without knowing anything about the database itself?
