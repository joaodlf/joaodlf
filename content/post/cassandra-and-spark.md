---
title: "Cassandra and Spark"
date: 2016-05-28
keywords:
- spark
- cassandra
---

On my previous post I went over [Running Apache Spark on a cluster](/running-apache-spark-on-a-cluster.html).
Spark can read and write from many data sources, including
[Apache Cassandra](https://cassandra.apache.org/).

Cassandra is a _distributed database management system_. It is a considered a _NoSQL_ database (the usage of such term is questionable,
albeit outside of the scope of this post).

Cassandra focuses on scalability and availability, therefore it misses features that are usually present in relational database solutions -
Features like Ad-Hoc query functionality such as SQL SUM or GROUP BY.

Spark can provide for such needs, as well as equip you with a very powerful data analysis tool - the **DataFrame**.

From the Spark [documentation](https://spark.apache.org/docs/latest/sql-programming-guide.html):

>A DataFrame is a distributed collection of data organized into named columns. It is conceptually equivalent to a table in a relational
database or a data frame in R/Python, but with richer optimizations under the hood. DataFrames can be constructed from a wide array of
sources such as: structured data files, tables in Hive, external databases, or existing RDDs.

It's safe to say the major inspiration for Sparks DataFrame is the [pandas](http://pandas.pydata.org/) 
library for Python. If you've ever worked with pandas, Spark operations should feel similar.

This post will focus on how to link to Cassandra via Spark, as well as perform DataFrame operations to aggregate the original data. This new
data will then be written back to Cassandra. All of this in Python.

### The Cassandra setup

We will need a Cassandra table (with some test data) to read from:

```sql
CREATE TABLE spark_test (
    day int,
    client_id int,
    traffic_source text,
    clicks counter,
    revenue counter,
    PRIMARY KEY ((day, client_id), traffic_source)
);

UPDATE spark_test SET clicks = clicks + 1, revenue = revenue + 30 WHERE day = 20160528 AND client_id = 1 AND traffic_source = 'Google';
UPDATE spark_test SET clicks = clicks + 1, revenue = revenue + 20 WHERE day = 20160528 AND client_id = 1 AND traffic_source = 'Facebook';
UPDATE spark_test SET clicks = clicks + 1, revenue = revenue + 10 WHERE day = 20160528 AND client_id = 1 AND traffic_source = 'Bing';
UPDATE spark_test SET clicks = clicks + 1, revenue = revenue + 10 WHERE day = 20160528 AND client_id = 2 AND traffic_source = 'Google';
UPDATE spark_test SET clicks = clicks + 1, revenue = revenue + 30 WHERE day = 20160528 AND client_id = 2 AND traffic_source = 'Facebook';
UPDATE spark_test SET clicks = clicks + 1, revenue = revenue + 10 WHERE day = 20160528 AND client_id = 3 AND traffic_source = 'Google';
UPDATE spark_test SET clicks = clicks + 1, revenue = revenue + 20 WHERE day = 20160528 AND client_id = 3 AND traffic_source = 'Bing';
```

As well as a second table to write to:

```sql
CREATE TABLE spark_test_traffic_sources (
    traffic_source text,
    clicks counter,
    revenue counter,
    PRIMARY KEY ((traffic_source))
);
```

### Linking to Cassandra

We begin by writing our initial Python script, which will import some of the available Spark classes:

```python
from pyspark import SparkConf, SparkContext
from pyspark.sql import SQLContext
import pyspark.sql.functions as func

# SQL functions will be used later...
import pyspark.sql.functions as func

# Configuration details
conf = SparkConf().setAppName("spark_cassandra_test")
sc = SparkContext(conf=conf)
sqlContext = SQLContext(sc)
    
# Load the DataFrame by linking it to a Cassandra table
df = sqlContext \
    .read \
    .format("org.apache.spark.sql.cassandra") \
    .options(table="spark_test", keyspace="tests") \
    .load()

# Show the top rows
df.show()
```

I'm saving the above code in a new file: **spark_cassandra.py** - If you are using PyCharm and OSX,
[here](https://stackoverflow.com/questions/34685905/how-to-link-pycharm-with-pyspark/36415945#36415945) is how you
can setup the IDE to use the Spark environment variables - After you've setup your work environment, working with PySpark should be a lot
more intuitive.

It's important to note that out of the box, Spark doesn't support reading from Cassandra. We will need to run the script using the
Spark Cassandra Connector package.

### Running the script

We launch our application via the **spark-submit** script. Before doing that though, it's important to go over some of the optional flags:

* **_--master_** : The master URL of your cluster.
* **_--packages_** : List of comma-delimited dependencies (this is where the Spark Cassandra Connector comes in).
* **_--executor-memory and --executor-cores_** : Heap size and number of cores to be used by your application, respectively. Some more
information on these settings can be found 
[here](http://blog.cloudera.com/blog/2015/03/how-to-tune-your-apache-spark-jobs-part-2/). Make sure you understand
these so you can benefit the most out of your hardware.
* **_PYSPARK_PYTHON_** : This isn't an actual flag, but it defines what Python binary executable to use. This comes in handy if you'd
like to specify your own virtualenv _(and you always should!)_
* **_SPARK_HOME_** : This is completely optional, it defines the location of Spark in your system. Needed if you need to run
Spark applications via cron jobs or bash scripts.

There are other flags and settings available, I consider the above as some of the most important ones - at least initially.

For the sake of running this simple example, I'll just focus on specifying my Master URL and the Spark Cassandra Connector package.
Since I'm using Spark 1.6.1 and Scala 2.10, my package version is **datastax:spark-cassandra-connector:1.6.0-M2-s_2.10**. You can see all
the package versions [here](https://spark-packages.org/package/datastax/spark-cassandra-connector), make sure to check
against your own versions of Spark and Scala.

Finally, we run the Spark application:

```bash
./bin/spark-submit --master <Spark Master URL>  --packages datastax:spark-cassandra-connector:1.6.0-M2-s_2.10 spark_cassandra.py
```

Once it finishes, the output should be:

    +--------+---------+--------------+------+-------+
    |     day|client_id|traffic_source|clicks|revenue|
    +--------+---------+--------------+------+-------+
    |20160528|        2|      Facebook|     1|     30|
    |20160528|        2|        Google|     1|     10|
    |20160528|        1|          Bing|     1|     10|
    |20160528|        1|      Facebook|     1|     20|
    |20160528|        1|        Google|     1|     30|
    |20160528|        3|          Bing|     1|     20|
    |20160528|        3|        Google|     1|     10|
    +--------+---------+--------------+------+-------+

### DataFrame operations

I'm particularly interested in grouping my traffic sources and summing up both clicks and revenue, independent of client.

Editing our original script,

```python
df = df\
    .filter(df["day"] == 20160528)\
    .groupBy("traffic_source")\
    .agg(func.sum("clicks").alias("clicks"), func.sum("revenue").alias("revenue"))\
    .show()
```

Which will in turn output,

    +--------------+------+-------+
    |traffic_source|clicks|revenue|
    +--------------+------+-------+
    |      Facebook|     2|     50|
    |          Bing|     2|     30|
    |        Google|     3|     50|
    +--------------+------+-------+

We have effectively run a SQL like GROUP BY and SUM operation using Spark!

### Writing to Cassandra

As long as the DataFrame matches the schema of our destination table, writing to Cassandra is straightforward.

Again, editing the original script and adding:

```python
df\
    .drop("day")\
    .drop("client_id")\
    .write\
    .format("org.apache.spark.sql.cassandra")\
    .mode("append")\
    .options(table="spark_test_traffic_sources", keyspace="test")\
    .save()
```

Notice that I drop the day and client_id columns to match up with the _spark_test_traffic_sources_ table schema.

After executing the application again, a quick check via cqlsh shows the outcome:

    cqlsh:tests> SELECT * FROM spark_test_traffic_sources ;

     traffic_source | clicks | revenue
    ----------------+--------+---------
               Bing |      2 |      30
             Google |      3 |      50
           Facebook |      2 |      50

_And that's it!_  We've managed to read, modify and write data back to Cassandra using Spark.

The full **spark_cassandra.py** script should be something like this:

```python
from pyspark import SparkConf, SparkContext
from pyspark.sql import SQLContext
import pyspark.sql.functions as func

# Configuration details
conf = SparkConf().setAppName("spark_cassandra_test")
sc = SparkContext(conf=conf)
sqlContext = SQLContext(sc)

# Load the DataFrame by linking it to a Cassandra table
df = sqlContext \
    .read \
    .format("org.apache.spark.sql.cassandra") \
    .options(table="spark_test", keyspace="test") \
    .load()

# Perform aggregate operations
df = df\
    .filter(df["day"] == 20160528)\
    .groupBy("traffic_source")\
    .agg(func.sum("clicks").alias("clicks"), func.sum("revenue").alias("revenue"))

# Write data to Cassandra
df\
    .drop("day")\
    .drop("client_id")\
    .write\
    .format("org.apache.spark.sql.cassandra")\
    .mode("append")\
    .options(table="spark_test_traffic_sources", keyspace="test")\
    .save()
```