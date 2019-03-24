---
title: "Why I'm excited about Apache Arrow"
date: 2016-09-11
keywords:
- Apache Arrow
- Cassandra
- Pandas
- Python
- NumpyProtocolHandler
---

>**Disclaimer:** After reading up the limited amount of information available about Apache Spark, I've 
drawn my own conclusions on what the project purposes to fix. If any of the information below is incorrect, 
shoot me an email or post a comment so that I can rectify any mistakes!

[Apache Arrow](https://arrow.apache.org/) aims to provide a common (In-Memory) data layer for storage 
and data analysis systems.

It's backed by several key developers of other major open source projects: Cassandra, HBase, Pandas, Spark, etc. 
Arrow has already been pushed as "Top-Level Project" status by
the [Apache Software Foundation](https://blogs.apache.org/foundation/entry/the_apache_software_foundation_announces87),
skipping the incubating status altogether.

Arrow isn't a new "product". It's not a database solution or data analysis tool. What Arrow proposes to do is **facilitate 
communication between the top open source projects in these fields.**

I've identified two important goals for the project:

* Provide an optimized In-Memory solution (**Columnar In-Memory**) for the systems you already know and use.
* Share the above solution between systems, solving a big performance issue in the data analysis world: 
**Serialization and deserialization of data** (expressed as "s/d" for the remainder of this post).

Projects like Cassandra and Pandas serve different purposes (storing and analysing data, respectfully) and were developed by different 
people, so it's safe to assume the In-Memory layer differs between projects. Currently there is no way for these projects to interact
directly - Data from Cassandra needs to be converted into a format Pandas can operate on. 

As it is now, there is only one way (that I know of!) to operate over Cassandra data in Pandas: the 
[NumpyProtocolHandler](https://docs.datastax.com/en/drivers/python/3.4/api/cassandra/protocol.html#faster-deserialization). 
Results from Cassandra are deserialized into NumPy Arrays for Pandas to operate on (Pandas was essentially built on Numpy).

This will change with Arrow. Both Cassandra and Pandas (as well as many other projects) have committed to integrate with Arrow,
meaning the 2 systems could share a common In-Memory solution - Thus defeating the need to s/d data between the two.

These are fantastic news for anyone that works with large datasets - It's one of toughest challenges I face professionally:
Speeding up data analysis and serving up information to our users as fast as possible. In a lot of these scenarios we have already
identified where we spend most o our time: **s/d!** I for one can't wait to see what Apache Arrow brings to the table. 







