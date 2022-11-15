---
layout: post
title:  "Hash Tables"
categories: CMU Intro To Databases
---

Note: These are notes from the CMU 15-445-645 (Spring 2022) lecture [7](https://www.youtube.com/watch?v=9yUlSabzVwQ)


Typical Hash Table implementation
- Implements an unordered associative array that maps keys to values
- Uses a hash function to compute an offset into the array for a given key
- Space: O(n) Time: O(1) avg O(n) worst
- For a database constants matter, so it is important to optimize the O(1)
- Often worth it to trade additional space to reduce the constant factor

### Hash Functions
- How to map a large key space into a smaller domain of integers (index into the array)
	- Lose ordering in the process
- Trade off between being fast and reducing collision rate
- Facebook XXHash is state of the art
- Others: CRC-64, MurmurHash, Google CityHash, Facebook XXHash, Google FarmHash
- Do not use cryptographic hash functions for databases, overhead is not worth it because the system does not require any of the beneficial properties

### Hashing Scheme
- How to handle key collisions after hashing
- Trade off between allocating a large hash table and additional instruction go get/put keys
	- Collisions require finding another place for inserting a key/value, or traversing alternative locations when the space is filled by a different key than is being queried

#### Static Hashing
Require the DBMS to know the number of elements it wants to store, or rebuild the entire backing structures when there is overflow to allow for additional data

None unique keys (joins)
- Separate Linked List: Store values in separate storage area for each key
- Redundant Keys: Store duplicate key entries together in the hash table
	- This is easy to implement so this is what most systems do
- Some systems guarantee uniqueness by concatenating record id with other identifying info to ensure there is no overlap
Linear Probe Hashing (Open Addressing)
- Resolve collisions by linearly searching for next free slot in the table
- To determine whether an elements is present, hash to a location and scan for the entry
- Must store the key in the first free slot
- Insertions and deletions are generalizations of lookups (do different operation when the item is found or you reach an empty slot)
- Deletions leave behind tombstones to prevent scans from stopping early, when a relevant key may be stored further in the array
- Inserts are free to overwrite tombstones (even if a version of a that entry exists later in the array because subsequent lookups will find the item overwriting the tombstone in subsequent scans)
Robin Hood Hashing
- Variant of linear probe hashing that steals slots for items that are close to their optimal slot to minimize the average distance of keys from their ideal slot
- Each key tracks the number of positions it is away from its optimal slot
- On insert, a key takes the slot of another key if the first key is farther ways from its optimal position than the second key
- Susceptible to flooding
- Generally not as efficient as basic linear probe hashing (so almost no system uses it)
Cuckoo Hashing
- Use multiple hash tables with different hash function seeds (same function)
	- On insert, check every table and pick anyone that has a free slot
	- If no table has a free slot, evict the element from one of them and then e-hash it and find a new open location for it
	- Look-ups and deletion are always O(1) because only one location per hash table is checked

#### Dynamic Hashing

Tables resize themselves on demand (per insert/delete, no big bang reconstruction)

Chained Hashing
- Maintain a linked list of buckets for each slot in the hash table (buckets can store more than one entry per node in the list)
- Resolve collisions by placing all elements with the same hash key in the same bucket
- To determine existence, hash and scan the linked list
- Insertions and deletions are a generalization of lookups just like with linear probe hashing
- Buckets are usually allocated to be the same as the database page size
Extendible Hashing
- Chained-hashing approach where we split buckets instead of letting the linked list grow forever
- Top level slot array:
	- maintains pointers to each bucket
	- should be length equal to a power of 2 and double in size when necessary
	- the size of the array acts a global bit mask to map hashed values to a slot index to determine which pointer to follow to a bucket
- Inserts/Lookups
	1. Hash the key
	2. Bitwise & with bit mask
	3. Follow pointer at that index to correct bucket
	4. Scan the bucket
	- ex: size = 8 -> Bit mask for 8 bit hash: 00000111 Hashed value: 10111101 -> 00000101. Lookup/Insert would follow the pointer at idx 5
- Multiple slot locations can point to the same bucket
	- This will happen when another bucket splits and causes a resize
- When a bucket overflows
	- Reshuffle the bucket entries
		- Not required if the split bucket was mapped to by multiple slot indices
		- Alter the bit mask & increasing the number of bits to examine on the hash
	- Data movement is localized to just the split chain
![Hash Tables (for indexing)](/assets/images/Hash%20Tables/Hash%20Tables%20(for%20indexing).svg)

Linear Hashing
- The hash table maintains a pointer that tracks the next bucket to split
- When any bucket overflows:
	- Insert the item into an overflow bucket for the bucket that it hashed to (if it is not the one that is to be split)
	- split the bucket at the pointer location, even if it is not the one that overflows
	- Double the size of the slot array of bucket pointers if necessary
		- Create an additional hash function (change the modulo by factor of 2 and use a different seed) that will be utilized by any slot idx that is before the split point in the array
			- hash<sub>1</sub>(key) = key % n, hash<sub>2</sub>(key) = key % 2n, ...
			- hash<sub>i</sub>(key) = key % 2<sup>i</sup> * n
	- Rebalance the bucket that has been split using the proper hash function
	- When looking for a value that hashes to a slot that is before the split pointer in the array, use the new hash function
		- Each split will

- Use multiple hashes to find the right bucket for a given key (after a bucket split)
	- Any key that hashes to a value that is before resize,
- Can use different overflow criterion:
	- Space utilization
	- Average Length of Overflow Chains
