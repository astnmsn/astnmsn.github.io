---
layout: post
title:  "Memory Management & Buffer Cache"
categories: CMU Intro To Databases
---

Note: These are notes from the CMU 15-445-645 (Spring 2022) lecture [6](https://www.youtube.com/watch?v=Y9H2HaRKOIw)

#### Managing the movement of data

Spatial Control
- Where to write pages on disk
- Keep data that is often accessed together, as physically close together as possible to take advantage of sequential performance of disks
Temporal Control
- When to move data into memory and when to flush it back out to disk
- Disk I/O is the main bottleneck of the system, so the goal is to reduce the number of operations that require interaction with the disk
- The system needs to be aware which process is updating memory, so it knows when it is safe to flush (dirty records) to disk
Execution engine should be unaware of this memory management, queries only require that the underlying storage system returns a pointer to the relevant data in memory (location in the buffer pool).

#### Buffer Pool

- Memory region organized as an array of fixed-size pages (Database page)
- An array is called a ***frame***
- When the database requests a page, an exact copy is read from disk and placed into one of these frames
- Dirty pages are buffered and not written to disk immediately (Write-Back Cache), to minimize the number of I/O operations
Page Table
- Keeps track of pages that are currently in memory
- Routes page id requests to a particular frame
- Also maintains additional metadata per page
	- Dirty Flag
	- Pin/Reference Counter
		- How many processes are actively utilizing the page
		- A value of true/>0 signifies that the Buffer Pool Manager should not evict the page from memory
- A latch is placed around an entry in the frame when it is being modified so that it cannot be altered by another thread

Aside: Locks vs Latches
- Lock: protects database's logical contents from other transactions
- Latch: protects critical sections of the system's internal data structure from other threads (mutex)

Page Directory
- The mapping from page ids to page locations in database files
- This structure is required for maintaining the coherency of the underlying data and must itself be persisted to disk
- This allows the system to reconstruct the mapping on restart
Page Table
- The mapping from page ids to the location of a copy of the pages that have been brought into buffer pool frames
- Because the buffer pool itself is ephemeral, the page table does not need to be persisted, and would not be reconstructed on a restart (although some systems do allow for this at the cost of some ongoing performance cost, it can improve performance at the early moments of a restart as failed queries are likely going to be retried)

#### Allocation Policies

Global Policies -> All active queries
Local Policies -> Allocate frames to specific queries without considering the behavior of other concurrent queries. This still requires support for sharing pages. This can have adverse effects on the performance of contending threads if the allotment of memory is not equal.

#### Buffer Pool Optimizations

Multiple Buffer Pools
- Per-database buffer pool
- Per-page type buffer pool
- Partitioning memory across multiple pools helps
	- reduce latch contention -> fewer queries per table means that the load associated with managing the latches is spread across multiple tables
	- improve locality
	- records must always map to the same pool so that there are never multiple instances of the page in separate buffer pools (coherency issues with determining if it is dirty)
	- can maintain different eviction policies for each pool
- Object id
	- Embed an object identifier in record ids and the maintain a mapping from objects to specific buffer pools
- Hashing
	- Hash the page id to select which buffer pool to access

Prefetching
- The system can also prefetch pages based on a query plan
	- Sequential Scans
		- bring in multiple sequential pages at once
		- can evict previous pages in the scan to prefetch these additional pages
	- Index Scans
		- can prefetch pages that follow sibling pointers at the leaves of a B+ tree (these would not be contiguous on disk)
	- Scan Sharing (Synchronized Scans)
		- Queries can reuse data retrieved from storage or operator computations (not the same as result cacheing at a higher layer)
		- Allow multiple queries to attach to a single cursor that scans a table
		- Queries do not have to be the same
		- Can share intermediate result
		- Query that is piggy-backing must fetch the data it missed
		- Prevents thrashing of buffer pool
		- Can result in different results for a query (LIMIT) -> but this is allowed in the relational model
	- Buffer Pool Bypass
		- The sequential scan operator will not store fetched pages in the buffer pool to avoid overhead
		- Memory is local to the running query
		- Works well if operator needs to read a large sequence of pages that are contiguous on disk
		- Can also be used for temporary data (sorting, joins)
		- Good if the data is not likely to be used by another query for a long time as it prevents cluttering the global memory pool with data that is unlikely to be used by another thread
	- OS Page Cache
		- Most disk operations go through the OS API
		- Unless the database system tell it not to, the OS maintains its own filesystem cache (aka page cache, buffer cache)
		- Most systems use direct I/O to bypass the OS's cache, preventing redundant copies and cache eviction policies that are unaware of the workload being run

Buffer Replacement Policies

When the database needs to free up a frame to make room for a new page, it must decide which page to evict from the buffer pool.

The system will need to utilize a heuristic to determine which entry is the least likely to be used next.

- Least Recently Used
	- Maintain timestamp along the entry for when it was last accessed
	- When deciding what page to evict, select the page with the oldest timestamp
	- Can keep the pages in sorted order to reduce the time to find that entry
- Clock
	- Approximation of LRU that utilizes a single reference bit
	- When a page is accessed, set the bit to 1
	- Organize the pages in a circular buffer. When determining which page to evict, iterate through the buffer, if a page's bit is set to one, zero it. If it is set to zero, then evict.
	- Less metadata to track than LRU. No need to maintain sort of entries by timestamp which makes it less expensive to calculate
Both of these options are suscpetible to sequential flooding. Sequential scans pollute the buffer pool with pages that are read once, and then not again for a very long time. In a workload with a high amount of sequential scans, the most recently used page is the most unneeded page.

- LRU-K
	- Track the history of last K references to each page as timestamps and compute the interval between subsequent accesses
	- The system will use this history to estimate the next time that page is likely to be accessed, chooses the one that has the timestamp that is the furthest in the future
	- If a page has not been accessed k times when performing the eviction, set the value to infinity (evict immediately)
- Localization
	- The database system choose which pages to evict on a per query basis. This minimizes the pollution of the buffer pool from each query
	- Ex: Postgres maintains a small ring buffer that is private to the query
- Priority Hints
	- The system know about the context of each page during the query execution
	- Can provide hints to the buffer pool on whether a page is important or not
	- Ex: Query that utilizes an index should set a preference to maintain pages in the cache for nodes that are higher in the B+ tree
- Dirty Pages
	- Fast Path: If a page is not dirty, the the DBMS can simply drop it
	- Slow Path: If the page is dirty, the system must flush it to disk to ensure that changes are persisted
	- This leads to a trade off between fast evictions and dirty writing pages that will not be read again in the future
	- Background writing
		- Periodically walk through the page table and write dirty pages to disk
		- At this point the system can either simply clear the dirty flag, or evict the page

Other memory Pools
- Sorting & Join buffers
- Query Caches
- Maintenance buffers
- Log buffers
- Dictionary caches
