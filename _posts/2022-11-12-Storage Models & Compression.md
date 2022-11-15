---
layout: post
title:  "Storage Models & Compression"
categories: CMU Intro To Databases
---

Note: These are notes from the CMU 15-445-645 (Spring 2022) lecture [5](https://www.youtube.com/watch?v=q4W5r3GR0OU)

### Database Workloads
- Online Transaction Processing (OLTP)
	- Fast operations that read and update a small amount of data at a time
	- These systems need to serve many concurrent queries, typically on a random set of data, often related to a single entity
	- Often, this data is generated through direct/deliberate user interaction in a system
- Online Analytical Processing (OLAP)
	- Complex queries that read a lot of data to compute aggregates
	- Queries are often run in batches to perform analysis on a large amount of data, or streaming to compute aggregates
	- Often utilized for data science/analysis and business intelligence
	- Single records are rarely of importance
	- Typically combined with ETL or ELT
- Hybrid Transaction + Analytical Processings (HTAP)
	- OLTP + OLAP together on the same database instance
	- Fast transactions + analytics in one
	- Solutions that attempt both are usually not as a good at either individual workload

### Relational Model

Does not specify how the database system must store tuples on disk or within pages

The type of workload the system is meant to serve should determine how to store tuples

![Storage Models](/assets/images/Storage%20Models/Storage%20Models%20%26%20Compression%202022-11-12%2011.48.24.excalidraw.svg)

N-Ary Storage Model
- Store the entire tuple contiguously within a page
- Fast inserts, updates, deletes as the entire operation touches a single contiguous portion of the disk
- Not good for scanning large portions of the table and or a small subset of attributes
- This is better for OLTP workloads
	- Queries often reference the entire tuple for a single entity (want all information about the record)
- Postgres, MySQL, Sqlite

Decomposition Storage Model (DSM)
- Store values for a single attribute of all tuples adjacently within pages
- Also known as Column Store
- Reduces waste disk i/o bringing in values that are unnecessary to serve query
- Better query processing and data compression
- Slow for point queries, inserts, updates, and deletes because it requires splitting and stitching together tuples to write and read (respectively) an entire tuple
- This is better for OLAP workloads
	- Queries often target a small number of attributes across a huge number of entities
- Clickhouse, Bigtable, DuckDB, SingleStore
- Tuple Identification
	- Fixed length offsets
		- Each value is the same length for an attribute
		- Can use simple arithmetic to calculate the offset of tuples value in within the attribute pages
		- Potential waste of space if you allow variable length tuples, especially if there is a high variance between the largest value size and the average value size
		- Most systems use this layout
	- Embedded tuple ids
		- Each value is stored with its tuple id in a column
		- Must maintain a lookup table to map the ids to the offset

### Compression

- I/O is the main bottleneck if the system fetches data from disk during query execution
- The DBMS can compress pages to increase the effective amount of data brought in per I/O operation
- This leads to a trade-off between the work needed to compress/decompress and the compression ratio
	- Compression reduces DRAM requirement
	- May decrease CPU costs during the execution
- Data sets tend to have highly skewed distributions for attribute values (most values look very similar)
- Data sets tend to have a high correlation between attributes of the same tuple
	- Some attributes are dependent upon others -> Zip code to city

Compression Algorithms:
1. Must produce fixed-length values
	- Exceptions for var-length data stored in a separate pool
2. Should allow query execution to postpone decompression for as long as possible (late materialization)
	- Ideally, operations can be performed on the compressed data itself, which is only possible if it maintains some characteristics of the underlying data (sort ordering, equality checks)
3. Must be a lossless schema
	- Any kind of lossy compression must be performed at the application level
	- Otherwise, database may produce incorrect results

Granularity
1. Block-level
	-  Compress a block of tuples for the same table
	- Naive version: gzip, LZO, LZ4, Zstd
		- Is opaque to the database, does not convey anything about the underlying data
		- Computational overhead, Compression v Decompression speed
		- MySql InnoDB
			- Pad compressed block to fit the Database page size
			- Operations that don't require reading tuples in the block can be appended to a mod log
			- Operations that do require reading tuples require full decompression of the page
			- Mod log must be flushed to the compressed page when it gets full
1. Tuple-level
	- Compress the contents of the entire tuple (NSM-only)
2. Attribute-level
	- Compress a single attribute within one tuple (overflow)
	- Can target multiple attributes for the same page
3. Column-level
	- Compress multiple values for one or more attributes for multiple tuples (DSM-only)
	- Ex:
		- Run Length Encoding
			- Compress runs of the same value in a single column into triplet: (Value, Start position, # of elements)
			- Requires the columns to be sorted intelligently to maximize the compression opportunities
			- For whichever column you are sorting on, you have to reflect that into the other columns in order to maintain matching offsets (NP complete problem)
			- Can still compute aggregations on the attribute (count, avg, max, min, etc)
			- With a poor sort, the data can actually grow in size (for very small run lengths)
		- Bit Packing Encoding
			- When values for an attribute are always less than the value's declared largest size, store them as a smaller data type
			- Must resize if we ever add a value that requires more bits than we have allocated
			- Mostly Encoding -> uses a special marker to indicate when a value exceeds largest size and then maintain a look-up table to store them
		- Bitmap Encoding
			- For low cardinality columns, store a separate bitmap for each unique value for an attribute where an offset in the vector corresponds to a tuple
				- The i'th position in the bitmap is the i'th tuple in the table
				- For an attribute with cardinality=4: value `[3] -> [0, 0, 1, 0]`
				- Typically segmented into chunks to avoid allocating large blocks of contiguous memory
			- Some systems provide bitmap indexes
			- Every time an application inserts an unseen value, the system must extend all the bitmaps
		- Delta Encoding
			- Record the difference between values that follow each other in the same column
			- Store the base value inline or in a separate lookup table
			- Only useful if the base is large and the variance between entries is small
			- Can be combined with other encodings (Run Length or Bit Packing)
		- Incremental Encoding
			- Type of delta encoding that avoids duplicating common prefixes/suffixes between consecutive tuples
			- This works best with sorted data
		- Dictionary Encoding
			- Build a data structure that maps variable-length values to a smaller integer identifier
			- Replace those values with their corresponding identifier in the dictionary data structure
				- The dictionary must support: Encode/Locate & Decode/Extract (should be fast)
				- Must also support range queries. This means the encoded values need to support the same collation as the original values (assign codes based on the sorted values)
			- Will do whole length replacement of a value with a single compressed representation
			- Does not break up a value into a combination of mapped representations
			- This is what separates it from a naive algorithm, which will encode any arbitrary segments of the entire page, without concern for the value boundaries
			- Queries replace values with the mapped (compressed representation) prior to execution
			- DISTINCT queries can only access the encoding map
			- Most widely used compression scheme

