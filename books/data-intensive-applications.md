# [Designing Data-Intensive Applications]

## Reliable, scalable, and maintainable applications

A data-intensive application is typically built from standard building blocks. They usually need to:
* Store data (_databases_)
* Speed up reads (_caches_)
* Search data (_search indexes_)
* Send a message to another process asynchronously (_stream processing_)
* Periodically crunch data (_batch processing_)

* **Reliability**. To work _correctly_ even in the face of _adversity_.
* **Scalability**. Reasonable ways of dealing with growth.
* **Maintainability**. Be able to work on it _productively_.

### Reliability

Typical expectations:
* Application performs the function the user expected
* Tolerate the user making mistakes
* Its performance is good
* The system prevents abuse

Systems that anticipate faults and can cope with them are called _fault-tolerant_ or _resilient_.

**A fault is usually defined as one component of the system deviating from its spec**, whereas _failure_ is when the system as a whole stops providing the required service to the user.

You should generally **prefer tolerating faults over preventing faults**.

* **Hardware faults**. Until recently, redundancy of hardware components was sufficient for most applications. As data volumes increase, more applications use a larger number of machines, proportionally increasing the rate of hardware faults. **There is a move towards systems that tolerate the loss of entire machines**. A system that tolerates machine failure can be patched one node at a time, without downtime of the entire system (_rolling upgrade_).
* **Software errors**. It is unlikely that a large number of hardware components will fail at the same time. Software errors are a systematic error within the system, they tend to cause many more system failures than uncorrelated hardware faults.
* **Human errors**. Humans are known to be unreliable. Configuration errors by operators are a leading cause of outages. You can make systems more reliable:
    - Minimising the opportunities for error, peg: with admin interfaces that make easy to do the "right thing" and discourage the "wrong thing".
    - Provide fully featured non-production _sandbox_ environments where people can explore and experiment safely.
    - Automated testing.
    - Quick and easy recovery from human error, fast to rollback configuration changes, roll out new code gradually and tools to recompute data.
    - Set up detailed and clear monitoring, such as performance metrics and error rates (_telemetry_).
    - Implement good management practices and training.

### Scalability

This is how do we cope with increased load. We need to succinctly describe the current load on the system; only then we can discuss growth questions.

---

#### Twitter example

Twitter main operations
- Post tweet: a user can publish a new message to their followers (4.6k req/sec, over 12k req/sec peak)
- Home timeline: a user can view tweets posted by the people they follow (300k req/sec)

Two ways of implementing those operations:
1. Posting a tweet simply inserts the new tweet into a global collection of tweets. When a user requests their home timeline, look up all the people they follow, find all the tweets for those users, and merge them (sorted by time). This could be done with a SQL `JOIN`.
2. Maintain a cache for each user's home timeline. When a user _posts a tweet_, look up all the people who follow that user, and insert the new tweet into each of their home timeline caches.

Approach 1, systems struggle to keep up with the load of home timeline queries. So the company switched to approach 2. The average rate of published tweets is almost two orders of magnitude lower than the rate of home timeline reads.

Downside of approach 2 is that posting a tweet now requires a lot of extra work. Some users have over 30 million followers. A single tweet may result in over 30 million writes to home timelines.

Twitter moved to an hybrid of both approaches. Tweets continue to be fanned out to home timelines but a small number of users with a very large number of followers are fetched separately and merged with that user's home timeline when it is read, like in approach 1.

---

#### Describing performance

What happens when the load increases:
* How is the performance affected?
* How much do you need to increase your resources?

In a batch processing system such as Hadoop, we usually care about _throughput_, or the number of records we can process per second.

> ##### Latency and response time
> The response time is what the client sees. Latency is the duration that a request is waiting to be handled.

It's common to see the _average_ response time of a service reported. However, the mean is not very good metric if you want to know your "typical" response time, it does not tell you how many users actually experienced that delay.

**Better to use percentiles.**
* _Median_ (_50th percentile_ or _p50_). Half of user requests are served in less than the median response time, and the other half take longer than the median
* Percentiles _95th_, _99th_ and _99.9th_ (_p95_, _p99_ and _p999_) are good to figure out how bad your outliners are.

Amazon describes response time requirements for internal services in terms of the 99.9th percentile because the customers with the slowest requests are often those who have the most data. The most valuable customers.

On the other hand, optimising for the 99.99th percentile would be too expensive.

_Service level objectives_ (SLOs) and _service level agreements_ (SLAs) are contracts that define the expected performance and availability of a service.
An SLA may state the median response time to be less than 200ms and a 99th percentile under 1s. **These metrics set expectations for clients of the service and allow customers to demand a refund if the SLA is not met.**

Queueing delays often account for large part of the response times at high percentiles. **It is important to measure times on the client side.**

When generating load artificially, the client needs to keep sending requests independently of the response time.

> ##### Percentiles in practice
> Calls in parallel, the end-user request still needs to wait for the slowest of the parallel calls to complete.
> The chance of getting a slow call increases if an end-user request requires multiple backend calls.

#### Approaches for coping with load

* _Scaling up_ or _vertical scaling_: Moving to a more powerful machine
* _Scaling out_ or _horizontal scaling_: Distributing the load across multiple smaller machines.
* _Elastic_ systems: Automatically add computing resources when detected load increase. Quite useful if load is unpredictable.

Distributing stateless services across multiple machines is fairly straightforward. Taking stateful data systems from a single node to a distributed setup can introduce a lot of complexity. Until recently it was common wisdom to keep your database on a single node.

### Maintainability

The majority of the cost of software is in its ongoing maintenance. There are three design principles for software systems:
* **Operability**. Make it easy for operation teams to keep the system running.
* **Simplicity**. Easy for new engineers to understand the system by removing as much complexity as possible.
* **Evolvability**. Make it easy for engineers to make changes to the system in the future.

#### Operability: making life easy for operations

A good operations team is responsible for
* Monitoring and quickly restoring service if it goes into bad state
* Tracking down the cause of problems
* Keeping software and platforms up to date
* Keeping tabs on how different systems affect each other
* Anticipating future problems
* Establishing good practices and tools for development
* Perform complex maintenance tasks, like platform migration
* Maintaining the security of the system
* Defining processes that make operations predictable
* Preserving the organisation's knowledge about the system

**Good operability means making routine tasks easy.**

#### Simplicity: managing complexity

When complexity makes maintenance hard, budget and schedules are often overrun. There is a greater risk of introducing bugs.

Making a system simpler means removing _accidental_ complexity, as non inherent in the problem that the software solves (as seen by users).

One of the best tools we have for removing accidental complexity is _abstraction_ that hides the implementation details behind clean and simple to understand APIs and facades.

#### Evolvability: making change easy

_Agile_ working patterns provide a framework for adapting to change.

---

* _Functional requirements_: what the application should do
* _Nonfunctional requirements_: general properties like security, reliability, compliance, scalability, compatibility and maintainability.

---

## Data models and query language

Most applications are built by layering one data model on top of another. Each layer hides the complexity of the layers below by providing a clean data model. These abstractions allow different groups of people to work effectively.

### Relational model vs document model

The roots of relational databases lie in _business data processing_, _transaction processing_ and _batch processing_.

The goal was to hide the implementation details behind a cleaner interface.

_Not Only SQL_ has a few driving forces:
* Greater scalability
* preference for free and open source software
* Specialised query optimisations
* Desire for a more dynamic and expressive data model

**With a SQL model, if data is stored in a relational tables, an awkward translation can happen between application code (objects, models) and database rows and column, this is called _impedance mismatch_. **

JSON model reduces the impedance mismatch and the lack of schema is often cited as an advantage.

JSON representation has better **_locality_** than the multi-table SQL schema. All the relevant information is in one place, and one query is sufficient.

In relational databases, it's normal to refer to rows in other tables by ID, because joins are easy. In document databases, joins are not needed for one-to-many tree structures, and support for joins is often weak.

If the database itself does not support joins, you have to emulate a join in application code by making multiple queries.

The most popular database for business data processing in the 1970s was the IBM's _Information Management System_ (IMS).

IMS used a _hierarchical model_ and like document databases worked well for one-to-many relationships, but it made many-to-,any relationships difficult, and it didn't support joins.

#### The network model

Standardised by a committee called the Conference on Data Systems Languages (CODASYL) model was a generalisation of the hierarchical model. In the tree structure of the hierarchical model, every record has exactly one parent, while in the network model, a record could have multiple parents.

The links between records are like pointers in a programming language. The only way of accessing a record was to follow a path from a root record called _access path_.

A query in CODASYL was performed by moving a cursor through the database by iterating over a list of records. If you didn't have a path to the data you wanted, you were in a difficult situation as it was difficult to make changes to an application's data model.

#### The relational model

By contrast, the relational model was a way to lay out all the data in the open" a relation (table) is simply a collection of tuples (rows), and that's it.

The query optimiser automatically decides which parts of the query to execute in which order, and which indexes to use (the access path).

The relational model thus made it much easier to add new features to applications.

---

**The main arguments in favour of the document data model are schema flexibility, better performance due to locality, and sometimes closer data structures to the ones used by the applications. The relation model counters by providing better support for joins, and many-to-one and many-to-many relationships.**

If the data in your application has a document-like structure, then it's probably a good idea to use a document model. The relational technique of _shredding_ can lead unnecessary complicated application code.

The poor support for joins in document databases may or may not be a problem.

If you application does use many-to-many relationships, the document model becomes less appealing. Joins can be emulated in application code by making multiple requests. Using the document model can lead to significantly more complex application code and worse performance.

#### Schema flexibility

Most document databases do not enforce any schema on the data in documents. Arbitrary keys and values can be added to a document, when reading, **clients have no guarantees as to what fields the documents may contain.**

Document databases are sometimes called _schemaless_, but maybe a more appropriate term is _schema-on-read_, in contrast to _schema-on-write_.

Schema-on-read is similar to dynamic (runtime) type checking, whereas schema-on-write is similar to static (compile-time) type checking.

The schema-on-read approach if the items on the collection don't have all the same structure (heterogeneous)
* Many different types of objects
* Data determined by external systems


#### Data locality for queries

If your application often needs to access the entire document, there is a performance advantage to this _storage locality_.

The database typically needs to load the entire document, even if you access only a small portion of it. On updates, the entire document usually needs to be rewritten, it is recommended that you keep documents fairly small.

#### Convergence of document and relational databases

PostgreSQL does support JSON documents. RethinkDB supports relational-like joins in its query language and some MongoDB drivers automatically resolve database references. Relational and document databases are becoming more similar over time.

### Query languages for data

SQL is a _declarative_ query language. In an _imperative language_, you tell the computer to perform certain operations in order.

In a declarative query language you just specify the pattern of the data you want, but not _how_ to achieve that goal.

A declarative query language hides implementation details of the database engine, making it possible for the database system to introduce performance improvements without requiring any changes to queries.

Declarative languages often lend themselves to parallel execution while imperative code is very hard to parallelise across multiple cores because it specifies instructions that must be performed in a particular order. Declarative languages specify only the pattern of the results, not the algorithm that is used to determine results.

#### Declarative queries on the web

In a web browser, using declarative CSS styling is much better than manipulating styles imperatively in JavaScript. Declarative languages like SQL turned out to be much better than imperative query APIs.

#### MapReduce querying

_MapReduce_ is a programming model for processing large amounts of data in bulk across many machines, popularised by Google.

Mongo offers a MapReduce solution.

```js
db.observations.mapReduce(
    function map() { 
        var year  = this.observationTimestamp.getFullYear();
        var month = this.observationTimestamp.getMonth() + 1;
        emit(year + "-" + month, this.numAnimals); 
    },
    function reduce(key, values) { 
        return Array.sum(values); 
    },
    {
        query: { family: "Sharks" }, 1
        out: "monthlySharkReport" 
    }
);
```

The `map` and `reduce` functions must be _pure_ functions, they cannot perform additional database queries and they must not have any side effects. These restrictions allow the database to run the functions anywhere, in any order, and rerun them on failure.

A usability problem with MapReduce is that you have to write two carefully coordinated functions. A declarative language offers more opportunities for a query optimiser to improve the performance of a query. For there reasons, MongoDB 2.2 added support for a declarative query language called _aggregation pipeline_

```js
db.observations.aggregate([
    { $match: { family: "Sharks" } },
    { $group: {
        _id: {
            year:  { $year:  "$observationTimestamp" },
            month: { $month: "$observationTimestamp" }
        },
        totalAnimals: { $sum: "$numAnimals" }
    } }
]);
```

### Graph-like data models

If many-to-many relationships are very common in your application, it becomes more natural to start modelling your data as a graph.

A graph consists of _vertices_ (_nodes_ or _entities_) and _edges_ (_relationships_ or _arcs_).

Well-known algorithms can operate on these graphs, like the shortest path between two points, or popularity of a web page.

There are several ways of structuring and querying the data. The _property graph_ model (implemented by Neo4j, Titan, and Infinite Graph) and the _triple-store_ model (implemented by Datomic, AllegroGraph, and others). There are also three declarative query languages for graphs: Cypher, SPARQL, and Datalog.

#### Property graphs

Each vertex consists of:
* Unique identifier
* Outgoing edges
* Incoming edges
* Collection of properties (key-value pairs)

Each edge consists of:
* Unique identifier
* Vertex at which the edge starts (_tail vertex_)
* Vertex at which the edge ends (_head vertex_)
* Label to describe the kind of relationship between the two vertices
* A collection of properties (key-value pairs)

Graphs provide a great deal of flexibility for data modelling. Graphs are good for evolvability.

---

* _Cypher_ is a declarative language for property graphs created by Neo4j
* Graph queries in SQL. In a relational database, you usually know in advance which joins you need in your query. In a graph query, the number if joins is not fixed in advance. In Cypher `:WITHIN*0...` expresses "follow a `WITHIN` edge, zero or more times" (like the `*` operator in a regular expression). This idea of variable-length traversal paths in a query can be expressed using something called _recursive common table expressions_ (the `WITH RECURSIVE` syntax).

---

#### Triple-stores and SPARQL

In a triple-store, all information is stored in the form of very simple three-part statements: _subject_, _predicate_, _object_ (peg: _Jim_, _likes_, _bananas_). A triple is equivalent to a vertex in graph.

#### The SPARQL query language

_SPARQL_ is a query language for triple-stores using the RDF data model.

#### The foundation: Datalog

_Datalog_ provides the foundation that later query languages build upon. Its model is similar to the triple-store model, generalised a bit. Instead of writing a triple (_subject_, _predicate_, _object_), we write as _predicate(subject, object)_.

We define _rules_ that tell the database about new predicates and rules can refer to other rules, just like functions can call other functions or recursively call themselves.

Rules can be combined and reused in different queries. It's less convenient for simple one-off queries, but it can cope better if your data is complex.

## Storage and retrieval

Databases need to do two things: store the data and give the data back to you.

### Data structures that power up your database

Many databases use a _log_, which is append-only data file. Real databases have more issues to deal with tho (concurrency control, reclaiming disk space so the log doesn't grow forever and handling errors and partially written records).

> A _log_ is an append-only sequence of records

In order to efficiently find the value for a particular key, we need a different data structure: an _index_. An index is an _additional_ structure that is derived from the primary data.

Well-chosen indexes speed up read queries but every index slows down writes. That's why databases don't index everything by default, but require you to choose indexes manually using your knowledge on typical query patterns.

#### Hash indexes

Key-value stores are quite similar to the _dictionary_ type (hash map or hash table).

Let's say our storage consists only of appending to a file. The simplest indexing strategy is to keep an in-memory hash map where every key is mapped to a byte offset in the data file. Whenever you append a new key-value pair to the file, you also update the hash map to reflect the offset of the data you just wrote.

Bitcask (the default storage engine in Riak) does it like that. The only requirement it has is that all the keys fit in the available RAM. Values can use more space than there is available in memory, since they can be loaded from disk.

A storage engine like Bitcask is well suited to situations where the value for each key is updated frequently. There are a lot of writes, but there are too many distinct keys, you have a large number of writes per key, but it's feasible to keep all keys in memory.

As we only ever append to a file, so how do we avoid eventually running out of disk space? **A good solution is to break the log into segments of certain size by closing the segment file when it reaches a certain size, and making subsequent writes to a new segment file. We can then perform _compaction_ on these segments.** Compaction means throwing away duplicate keys in the log, and keeping only the most recent update for each key.

We can also merge several segments together at the same time as performing the compaction. Segments are never modified after they have been written, so the merged segment is written to a new file. Merging and compaction of frozen segments can be done in a background thread. After the merging process is complete, we switch read requests to use the new merged segment instead of the old segments, and the old segment files can simply be deleted.

Each segment now has its own in-memory hash table, mapping keys to file offsets. In order to find a value for a key, we first check the most recent segment hash map; if the key is not present we check the second-most recent segment and so on. The merging process keeps the number of segments small, so lookups don't need to check many hash maps.

Some issues that are important in a real implementation:
* File format. It is simpler to use binary format.
* Deleting records. Append special deletion record to the data file (_tombstone_) that tells the merging process to discard previous values.
* Crash recovery. If restarted, the in-memory hash maps are lost. You can recover from reading each segment but that would take long time. Bitcask speeds up recovery by storing a snapshot of each segment hash map on disk.
* Partially written records. The database may crash at any time. Bitcask includes checksums allowing corrupted parts of the log to be detected and ignored.
* Concurrency control. As writes are appended to the log in a strictly sequential order, a common implementation is to have a single writer thread. Segments are immutable, so they can be read concurrently by multiple threads.

Append-only design turns out to be good for several reasons:
* Appending and segment merging are sequential write operations, much faster than random writes, especially on magnetic spinning-disks.
* Concurrency and crash recovery are much simpler.
* Merging old segments avoids files getting fragmented over time.

Hash table has its limitations too:
* The hash table must fit in memory. It is difficult to make an on-disk hash map perform well.
* Range queries are not efficient.

#### SSTables and LSM-Trees

We introduce a new requirement to segment files: we require that the sequence of key-value pairs is _sorted by key_.

We call this _Sorted String Table_, or _SSTable_. We require that each key only appears once within each merged segment file (compaction already ensures that). SSTables have few big advantages over log segments with hash indexes
1. **Merging segments is simple and efficient** (we can use algorithms like _mergesort_). When multiple segments contain the same key, we can keep the value from the most recent segment and discard the values in older segments.
2. **You no longer need to keep an index of all the keys in memory.** For a key like `handiwork`, when you know the offsets for the keys `handback` and `handsome`, you know `handiwork` must appear between those two. You can jump to the offset for `handback` and scan from there until you find `handiwork`, if not, the key is not present. You still need an in-memory index to tell you the offsets for some of the keys. One key for every few kilobytes of segment file is sufficient.
3. Since read requests need to scan over several key-value pairs in the requested range anyway, **it is possible to group those records into a block and compress it** before writing it to disk.

How do we get the data sorted in the first place? With red-black trees or AVL trees, you can insert keys in any order and read them back in sorted order.
* When a write comes in, add it to an in-memory balanced tree structure (_memtable_).
* When the memtable gets bigger than some threshold (megabytes), write it out to disk as an SSTable file. Writes can continue to a new memtable instance.
* On a read request, try to find the key in the memtable, then in the most recent on-disk segment, then in the next-older segment, etc.
* From time to time, run merging and compaction in the background to discard overwritten and deleted values.

If the database crashes, the most recent writes are lost. We can keep a separate log on disk to which every write is immediately appended. That log is not in sorted order, but that doesn't matter, because its only purpose is to restore the memtable after crash. Every time the memtable is written out to an SSTable, the log can be discarded.

**Storage engines that are based on this principle of merging and compacting sorted files are often called LSM structure engines (Log Structure Merge-Tree).**

Lucene, an indexing engine for full-text search used by Elasticsearch and Solr, uses a similar method for storing its _term dictionary_.

LSM-tree algorithm can be slow when looking up keys that don't exist in the database. To optimise this, storage engines often use additional _Bloom filters_ (a memory-efficient data structure for approximating the contents of a set).

There are also different strategies to determine the order and timing of how SSTables are compacted and merged. Mainly two _size-tiered_ and _leveled_ compaction. LevelDB and RocksDB use leveled compaction, HBase use size-tiered, and Cassandra supports both. In size-tiered compaction, newer and smaller SSTables are successively merged into older and larger SSTables. In leveled compaction, the key range is split up into smaller SSTables and older data is moved into separate "levels", which allows the compaction to use less disk space.

#### B-trees

This is the most widely used indexing structure. B-tress keep key-value pairs sorted by key, which allows efficient key-value lookups and range queries.

The log-structured indexes break the database down into variable-size _segments_ typically several megabytes or more. B-trees break the database down into fixed-size _blocks_ or _pages_, traditionally 4KB.

One page is designated as the _root_ and you start from there. The page contains several keys and references to child pages.

If you want to update the value for an existing key in a B-tree, you search for the leaf page containing that key, change the value in that page, and write the page back to disk. If you want to add new key, find the page and add it to the page. If there isn't enough free space in the page to accommodate the new key, it is split in two half-full pages, and the parent page is updated to account for the new subdivision of key ranges.

Trees remain _balanced_. A B-tree with _n_ keys always has a depth of _O_(log _n_).

The basic underlying write operation of a B-tree is to overwrite a page on disk with new data. It is assumed that the overwrite does not change the location of the page, all references to that page remain intact. This is a big contrast to log-structured indexes such as LSM-trees, which only append to files.

Some operations require several different pages to be overwritten. When you split a page, you need to write the two pages that were split, and also overwrite their parent. If the database crashes after only some of the pages have been written, you end up with a corrupted index.

It is common to include an additional data structure on disk: a _write-ahead log_ (WAL, also know as the _redo log_).

Careful concurrency control is required if multiple threads are going to access, typically done protecting the tree internal data structures with _latches_ (lightweight locks).

#### B-trees and LSM-trees

LSM-trees are typically faster for writes, whereas B-trees are thought to be faster for reads. Reads are typically slower on LSM-tress as they have to check several different data structures and SSTables at different stages of compaction.

Advantages of LSM-trees:
* LSM-trees are typically able to sustain higher write throughput than B-trees, party because they sometimes have lower write amplification: a write to the database results in multiple writes to disk. The more a storage engine writes to disk, the fewer writes per second it can handle.
* LSM-trees can be compressed better, and thus often produce smaller files on disk than B-trees. B-trees tend to leave disk space unused due to fragmentation.

Downsides of LSM-trees:
* Compaction process can sometimes interfere with the performance of ongoing reads and writes. B-trees can be more predictable. The bigger the database, the the more disk bandwidth is required for compaction. Compaction cannot keep up with the rate of incoming writes, if not configured properly you can run out of disk space.
* On B-trees, each key exists in exactly one place in the index. This offers strong transactional semantics. Transaction isolation is implemented using locks on ranges of keys, and in a B-tree index, those locks can be directly attached to the tree.

#### Other indexing structures

We've only discussed key-value indexes, which are like _primary key_ index. There are also _secondary indexes_.

A secondary index can be easily constructed from a key-value index. The main difference is that in a secondary index, the indexed values are not necessarily unique. There are two ways of doing this: making each value in the index a list of matching row identifiers or by making a each entry unique by appending a row identifier to it.

#### Full-text search and fuzzy indexes

Indexes don't allow you to search for _similar_ keys, such as misspelled words. Such _fuzzy_ querying requires different techniques.

Full-text search engines allow synonyms, grammatical variations, occurrences of words near each other.

Lucene uses SSTable-like structure for its term dictionary. Lucene, the in-memory index is a finite state automaton, similar to a _trie_.

#### Keeping everything in memory

Disks have two significant advantages: they are durable, and they have lower cost per gigabyte than RAM.

It's quite feasible to keep them entirely in memory, this has lead to _in-memory_ databases.

Key-value stores, such as Memcached are intended for cache only, it's acceptable for data to be lost if the machine is restarted. Other in-memory databases aim for durability, with special hardware, writing a log of changes to disk, writing periodic snapshots to disk or by replicating in-memory sate to other machines.

When an in-memory database is restarted, it needs to reload its state, either from disk or over the network from a replica. The disk is merely used as an append-only log for durability, and reads are served entirely from memory.

Products such as VoltDB, MemSQL, and Oracle TimesTime are in-memory databases. Redis and Couchbase provide weak durability.

In-memory databases can be faster because they can avoid the overheads of encoding in-memory data structures in a form that can be written to disk.

Another interesting area is that in-memory databases may provide data models that are difficult to implement with disk-based indexes.

### Transaction processing or analytics?

A _transaction_ is a group of reads and writes that form a logical unit, this pattern became known as _online transaction processing_ (OLTP).

_Data analytics_ has very different access patterns. A query would need to scan over a huge number of records, only reading a few columns per record, and calculates aggregate statistics.

These queries are often written by business analysts, and fed into reports. This pattern became known for _online analytics processing_ (OLAP).


#### Data warehousing

A _data warehouse_ is a separate database that analysts can query to their heart's content without affecting OLTP operations. It contains read-only copy of the dat in all various OLTP systems in the company. Data is extracted out of OLTP databases (through periodic data dump or a continuous stream of update), transformed into an analysis-friendly schema, cleaned up, and then loaded into the data warehouse (process _Extract-Transform-Load_ or ETL).

A data warehouse is most commonly relational, but the internals of the systems can look quite different.

Amazon RedShift is hosted version of ParAccel. Apache Hive, Spark SQL, Cloudera Impala, Facebook Presto, Apache Tajo, and Apache Drill. Some of them are based on ideas from Google's Dremel.

Data warehouses are used in fairly formulaic style known as a _star schema_.

Facts are captured as individual events, because this allows maximum flexibility of analysis later. The fact table can become extremely large.

Dimensions represent the _who_, _what_, _where_, _when_, _how_ and _why_ of the event.

The name "star schema" comes from the fact than when the table relationships are visualised, the fact table is in the middle, surrounded by its dimension tables, like the rays of a star.

Fact tables often have over 100 columns, sometimes several hundred. Dimension tables can also be very wide.

### Column-oriented storage

In a row-oriented storage engine, when you do a query that filters on a specific field, the engine will load all those rows with all their fields into memory, parse them and filter out the ones that don't meet the requirement. This can take a long time.

_Column-oriented storage_ is simple: don't store all the values from one row together, but store all values from each _column_ together instead. If each column is stored in a separate file, a query only needs to read and parse those columns that are used in a query, which can save a lot of work.

Column-oriented storage often lends itself very well to compression as the sequences of values for each column look quite repetitive, which is a good sign for compression. A technique that is particularly effective in data warehouses is _bitmap encoding_.

Bitmap indexes are well suited for all kinds of queries that are common in a data warehouse.

> Cassandra and HBase have a concept of _column families_, which they inherited from Bigtable.

Besides reducing the volume of data that needs to be loaded from disk, column-oriented storage layouts are also good for making efficient use of CPU cycles (_vectorised processing_).

**Column-oriented storage, compression, and sorting helps to make read queries faster and make sense in data warehouses, where most of the load consist on large read-only queries run by analysts. The downside is that writes are more difficult.**

An update-in-place approach, like B-tree use, is not possible with compressed columns. If you insert a row in the middle of a sorted table, you would most likely have to rewrite all column files.

It's worth mentioning _materialised aggregates_ as some cache of the counts ant the sums that queries use most often. A way of creating such a cache is with a _materialised view_, on a relational model this is usually called a _virtual view_: a table-like object whose contents are the results of some query. A materialised view is an actual copy of the query results, written in disk, whereas a virtual view is just a shortcut for writing queries.

When the underlying data changes, a materialised view needs to be updated, because it is denormalised copy of the data. Database can do it automatically, but writes would become more expensive.

A common special case of a materialised view is know as a _data cube_ or _OLAP cube_, a grid of aggregates grouped by different dimensions.

## Encoding and evolution

Change to an application's features also requires a change to data it stores.

Relational databases conforms to one schema although that schema can be changed, there is one schema in force at any point in time. **Schema-on-read (or schemaless) contain a mixture of older and newer data formats.**

In large applications changes don't happen instantaneously. You want to perform a _rolling upgrade_ and deploy a new version to a few nodes at a time, gradually working your way through all the nodes without service downtime.

Old and new versions of the code, and old and new data formats, may potentially all coexist. We need to maintain compatibility in both directions
* Backward compatibility, newer code can read data that was written by older code.
* Forward compatibility, older code can read data that was written by newer code.

### Formats for encoding data

Two different representations:
* In memory
* When you want to write data to a file or send it over the network, you have to encode it

Thus, you need a translation between the two representations. In-memory representation to byte sequence is called _encoding_ (_serialisation_ or _marshalling_), and the reverse is called _decoding_ (_parsing_, _deserialisation_ or _unmarshalling_).

Programming languages come with built-in support for encoding in-memory objects into byte sequences, but is usually a bad idea to use them. Precisely because of a few problems.
* Often tied to a particular programming language.
* The decoding process needs to be able to instantiate arbitrary classes and this is frequently a security hole.
* Versioning
* Efficiency

Standardised encodings can be written and read by many programming languages.

JSON, XML, and CSV are human-readable and popular specially as data interchange formats, but they have some subtle problems:
* Ambiguity around the encoding of numbers and dealing with large numbers
* Support of Unicode character strings, but no support for binary strings. People get around this by encoding binary data as Base64, which increases the data size by 33%.
* There is optional schema support for both XML and JSON
* CSV does not have any schema

#### Binary encoding

JSON is less verbose than XML, but both still use a lot of space compared to binary formats. There are binary encodings for JSON (MesagePack, BSON, BJSON, UBJSON, BISON and Smile), similar thing for XML (WBXML and Fast Infoset).

**Apache Thrift and Protocol Buffers (protobuf) are binary encoding libraries.**

Thrift offers two different protocols:
* **BinaryProtocol**, there are no field names like `userName`, `favouriteNumber`. Instead the data contains _field tags_, which are numbers (`1`, `2`)
* **CompactProtocol**, which is equivalent to BinaryProtocol but it packs the same information in less space. It packs the field type and the tag number into the same byte.

Protocol Buffers are very similar to Thrift's CompactProtocol, bit packing is a bit different and that might allow smaller compression.

Schemas inevitable need to change over time (_schema evolution_), how do Thrift and Protocol Buffers handle schema changes while keeping backward and forward compatibility changes?

* **Forward compatible support**. As with new fields you add new tag numbers, old code trying to read new code, it can simply ignore not recognised tags.
* **Backwards compatible support**. As long as each field has a unique tag number, new code can always read old data. Every field you add after initial deployment of schema must be optional or have a default value.

Removing fields is just like adding a field with backward and forward concerns reversed. You can only remove a field that is optional, and you can never use the same tag again.

What about changing the data type of a field? There is a risk that values will lose precision or get truncated.

##### Avro

Apache Avro is another binary format that has two schema languages, one intended for human editing (Avro IDL), and one (based on JSON) that is more easily machine-readable.

You go through the fields in the order they appear in the schema and use the schema to tell you the datatype of each field. Any mismatch in the schema between the reader and the writer would mean incorrectly decoded data.

What about schema evolution? When an application wants to encode some data, it encodes the data using whatever version of the schema it knows (_writer's schema_).

When an application wants to decode some data, it is expecting the data to be in some schema (_reader's schema_).

In Avro the writer's schema and the reader's schema _don't have to be the same_. The Avro library resolves the differences by looking at the writer's schema and the reader's schema.

Forward compatibility means you can have a new version of the schema as writer and an old version of the schema as reader. Conversely, backward compatibility means that you can have a new version of the schema as reader and an old version as writer.

To maintain compatibility, you may only add or remove a field that has a default value.

If you were to add a field that has no default value, new readers wouldn't be able to read data written by old writers.

Changing the datatype of a field is possible, provided that Avro can convert the type. Changing the name of a field is tricky (backward compatible but not forward compatible).

The schema is identified encoded in the data. In a large file with lots of records, the writer of the file can just include the schema at the beginning of the file. On a database with individually written records, you cannot assume all the records will have the same schema, so you have to include a version number at the beginning of every encoded record. While sending records over the network, you can negotiate the schema version on connection setup.

Avro is friendlier to _dynamically generated schemas_ (dumping into a file the database). You can fairly easily generate an Avro schema in JSON.

If the database schema changes, you can just generate a new Avro schema for the updated database schema and export data in the new Avro schema.

By contrast with Thrift and Protocol Buffers, every time the database schema changes, you would have to manually update the mappings from database column names to field tags.

---

Although textual formats such as JSON, XML and CSV are widespread, binary encodings based on schemas are also a viable option. As they have nice properties:
* Can be much more compact, since they can omit field names from the encoded data.
* Schema is a valuable form of documentation, required for decoding, you can be sure it is up to date.
* Database of schemas allows you to check forward and backward compatibility changes.
* Generate code from the schema is useful, since it enables type checking at compile time.

### Modes of dataflow

Different process on how data flows between processes

#### Via databases

The process that writes to the database encodes the data, and the process that reads from the database decodes it.

A value in the database may be written by a _newer_ version of the code, and subsequently read by an _older_ version of the code that is still running.

When a new version of your application is deployed, you may entirely replace the old version with the new version within a few minutes. The same is not true in databases, the five-year-old data will still be there, in the original encoding, unless you have explicitly rewritten it. _Data outlives code_.

Rewriting (_migrating_) is expensive, most relational databases allow simple schema changes, such as adding a new column with a `null` default value without rewriting existing data. When an old row is read, the database fills in `null`s for any columns that are missing.

#### Via service calls

You have processes that need to communicate over a network of _clients_ and _servers_.

Services are similar to databases, each service should be owned by one team. and that team should be able to release versions of the service frequently, without having to coordinate with other teams. We should expect old and new versions of servers and clients to be running at the same time.

_Remote procedure calls_ (RPC) tries to make a request to a remote network service look the same as calling a function or method in your programming language, it seems convenient at first but the approach is flawed:
* A network request is unpredictable
* A network request it may return without a result, due a _timeout_
* Retrying will cause the action to be performed multiple times, unless you build a mechanism for deduplication (_idempotence_).
* A network request is much slower than a function call, and its latency is wildly variable.
* Parameters need to be encoded into a sequence of bytes that can be sent over the network and becomes problematic with larger objects.
* The RPC framework must translate datatypes from one language to another, not all languages have the same types.

**There is no point trying to make a remote service look too much like a local object in your programming language, because it's a fundamentally different thing.**

New generation of RPC frameworks are more explicit about the fact that a remote request is different from a local function call. Fiangle and Rest.li use _features_ (_promises_) to encapsulate asyncrhonous actions.

RESTful API has some significant advantages like being good for experimentation and debugging.

REST seems to be the predominant style for public APIs. The main focus of RPC frameworks is on requests between services owned by the same organisation, typically within the same datacenter.

#### Via asynchronous message passing

In an _asynchronous message-passing_ systems, a client's request (usually called a _message_) is delivered to another process with low latency. The message goes via an intermediary called a _message broker_ (_message queue_ or _message-oriented middleware_) which stores the message temporarily. This has several advantages compared to direct RPC:
* It can act as a buffer if the recipient is unavailable or overloaded
* It can automatically redeliver messages to a process that has crashed and prevent messages from being lost
* It avoids the sender needing to know the IP address and port number of the recipient (useful in a cloud environment)
* It allows one message to be sent to several recipients
* **Decouples the sender from the recipient**

The communication happens only in one direction. The sender doesn't wait for the message to be delivered, but simply sends it and then forgets about it (_asynchronous_).

Open source implementations for message brokers are RabbitMQ, ActiveMQ, HornetQ, NATS, and Apache Kafka.

One process sends a message to a named _queue_ or _topic_ and the broker ensures that the message is delivered to one or more _consumers_ or _subscribers_ to that queue or topic.

Message brokers typically don't enforce a particular data model, you can use any encoding format.

An _actor model_ is a programming model for concurrency in a single process. Rather than dealing with threads (and their complications), logic is encapsulated in _actors_. Each actor typically represent one client or entity, it may have some local state, and it communicates with other actors by sending and receiving asynchronous messages. Message deliver is not guaranteed. Since each actor processes only one message at a time, it doesn't need to worry about threads.

In _distributed actor frameworks_, this programming model is used to scale an application across multiple nodes. It basically integrates a message broker and the actor model into a single framework.

* _Akka_ uses Java's built-in serialisation by default, which does not provide forward or backward compatibility. You can replace it with something like Protocol Buffers and the ability to do rolling upgrades.
* _Orleans_ by default uses custom data encoding format that does not support rolling upgrade deployments.
* In _Erlang OTP_ it is surprisingly hard to make changes to record schemas.

---

What happens if multiple machines are involved in storage and retrieval of data?

Reasons for distribute a database across multiple machines:
* Scalability
* Fault tolerance/high availability
* Latency, having servers at various locations worldwide


# PART II : Distributed Data

* Vertical scaling approach can have some problems : 
	* Not easily scalable (bottlenecks between components can occurs)
	* Geographicaly limited

---
### Replication versus partitioning
* **Replication** is keeping a copy of the data on different nodes, provides redundancy.
* **Partitioning** is splitting the database into smaller subset called partitions (also called **shards**)

---
## Chapitre 5 : Réplication

Plusieurs cas d'utilisation de la réplication :

- Garder les données géographiquement proches des utilisateurs.
- Permettre au système de continuer à fonctionner en cas de défaillance d'un nœud.
- Améliorer l'évolutivité des opérations de lecture.

### Leaders et Followers

Chaque nœud qui stocke une copie des données de la base de données est appelé une **réplica**. Pour que la réplication fonctionne correctement, chaque écriture doit être traitée. La solution la plus courante est la **réplication basée sur un leader**.

1. Le leader prend la responsabilité de gérer les écritures sur la base de données.
2. Les followers reçoivent le journal de réplication du leader et mettent à jour leur base de données.
3. Les followers ou le leader peuvent être interrogés pour lire les données.
![[Pasted image 20240917134959.png]]

### Synchronous versus asynchronous replication
![[Pasted image 20240918131157.png]]

- Leader avec un follower synchrone et un follower asynchrone : **semi-synchronous replication**.
    - L'avantage d'une réplication entièrement synchrone est que si le leader échoue, un follower peut rapidement être élu leader tout en ayant toutes les données dans un état cohérent. L'inconvénient est que si un follower échoue, l'écriture ne peut pas être traitée.
    - Une réplication entièrement asynchrone peut continuer à traiter les écritures même si le leader échoue, mais l'inconvénient est que la base de données peut se retrouver dans un état incohérent dans ce cas.

### Setting up new followers

- Dans la plupart des bases de données, la procédure consiste à :
    - Prendre un snapshot du leader à un moment donné.
    - Le copier sur le nœud follower.
    - Le follower se connecte au leader et demande toutes les modifications de données à partir de la position dans le journal de réplication du leader et du snapshot.

### Handling Node Outages

Il existe différentes techniques pour récupérer un nœud après une panne :

- **Follower failure - Catch-up recovery** : Processus expliqué dans la section "Setting up new followers".

- **Leader failure - Failover** :
    - Premièrement, il faut détecter que le leader a échoué (par exemple via un healthcheck entre les nœuds).
    - Ensuite, il faut choisir un nouveau leader (généralement le nœud le plus à jour).
    - Enfin, reconfigurer le système pour utiliser le nouveau leader.

### Implementation of replication logs

- **Statement based replication** : Chaque instruction modifiant les données (SELECT, UPDATE...) est envoyée au follower.
	- Peut poser des problèmes complexes pouvant casser la réplication : utilisation de méthodes non déterministes telles que RAND() ou NOW(), exécution des instructions dans un ordre différent, effets secondaires des triggers, etc.
- **Write-ahead-log shipping** : Mécanisme de (log) en append-only contenant toutes les opérations d'écriture sous forme d'octets (utilisé par **Postgres** et **Oracle**).
    - Niveau d'intrication bas : un WAL décrit quels octets ont été modifiés dans quels blocs de disque. Si le format de stockage change, il sera impossible d'utiliser la version précédente du WAL.
- **Logical (row-based) log replication** : Consiste à découpler le format du journal des mécanismes internes du moteur de stockage. On l'appelle ça **un logical log**, il contient une séquence d'enregistrements décrivant les écritures dans la base de données. Il offre une meilleure compatibilité en arrière grâce à ce découplage en termes de format.

### Problems with replication lags

La réplication basée sur un leader est un modèle courant pour les applications web : il y a généralement beaucoup plus de lectures que d'écritures, donc pour améliorer les performances de lecture, il suffit d'ajouter plus de followers. Des problèmes peuvent survenir lorsque les followers prennent du retard par rapport au leader en termes de données répliquées, ce qui est généralement dû à un **replication lag** : cela symbolise le délai de réplication entre les nœuds.

#### Reading your own writes
![[Pasted image 20240920113230.png]]

Il peut arriver qu'un utilisateur de l'application mette à jour ses données et voie des informations non à jour s'il fait une demande peu de temps après avoir soumis une modification : il peut lire des données incohérentes d'un follower si celui-ci est en retard par rapport au leader en termes de réplication. Quelques astuces peuvent prévenir ce problème :

- Lorsqu'on lit une donnée que l'utilisateur pourrait avoir modifiée, il faut la lire directement depuis le leader.
- On pourrait suivre l'heure de la dernière mise à jour. Si elle dépasse une minute, on peut lire depuis le follower, sinon, depuis le leader.
- On peut se souvenir du timestamp envoyé par le client pour comparer avec le timestamp de la dernière mise à jour enregistrée dans la réplica.

#### Monotonic Reads
![[Pasted image 20240920114512.png]]
Si un utilisateur requête plusieurs fois la même data, mais qu'il la requête depuis différents replicas, il peut avoir l'impression de "remonter le temps" dans le sens où si un replica possède un plus gros lag qu'un autre, l'opération de read peut être inconsistante.

C'est en cela qu'on a besoin de **monotonic read** : c'est le fait qu'un user fasse ses opérations de lecture à chaque fois sur le même replica. 

#### Consistent Prefix Reads
![[Pasted image 20240920115053.png]]
Le problème énoncé dans le schéma est le suivant : si une partition est répliquée plus lentement qu'une autre, une utilisateur pourrait avoir la réponse à la question avant la question en elle même.

On peut ici utiliser le mécanisme de **consistent prefix reads** qui garanti que si une séquences d'écritures en base de données se passent dans un certain ordre alors la lecture devra se faire dans le même ordre.

## Multi-Leader Replication
Une réplication multi-leader représente une complexité supplémentaire, mais peut être utile dans certains usecases : 

### Multi-datacenter operation
Dans une configuration multi-datacenter, on peut avoir un leader pour chaque datacenter, comme le présente ce schéma : 
![[Pasted image 20240921101258.png]]
Plusieurs avantages sont présents dans cette configuration :
* **Performance** : La possibilité d'avoir un leader par data-center permet d'éviter la latence au cas où le data-center est éloigné de l'utilisateur.
* **Tolerance à la panne** : Ici, si l'un des leaders échoue, les autres data-centers pourront continuer à encaisser la charge.
* **Tolerance aux problèmes réseau**

### Handling Write conflict
![[Pasted image 20240921101926.png]]

Un conflit d'écriture peut avoir lieu lorsque deux utilisateurs mettent à jour simultanément une même donnée dans le cadre d'une architecture multi leader.
Dans ce cas, comment résoudre ce conflit ? 

La stratégie la plus simple est de faire en sorte de les éviter, par exemple en routant spécifiquement certaines requêtes vers un leader designé.

D'autres stratégies existent cependant :
- Donner à chaque writes un ID. C'est l'ID le plus haut qui sera retenu.
- Se baser le l'ID du replica, le plus haut ID est retenu.
- Persister le conflit pour le résoudre plus tard.

### Multi-Leader Replication Topologies
![[Pasted image 20240921105230.png]]

Il existe plusieurs types de topologies de réplication différentes, chacune avec des avantages et inconvénients, par exemple pour la all-to-all topology :
![[Pasted image 20240921110302.png]]

On constate ici un problème similaire à celui qu'on a pu observer lors du chapitre "Consistent Prefix Reads", la mise à jour dépend de l'insertion, donc on doit garder une logique d'ordre. 

En bref : la réplication multi-leader en général, c'est pas terrible.