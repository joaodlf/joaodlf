---
title: "Running Apache Spark on a cluster"
date: 2016-05-03
keywords:
- Spark
- Spark cluster
---

Apache [Spark](https://spark.apache.org/) is a general-purpose data processing and analysis engine.

On the surface, it helps developers working with large data sets by providing easy to use libraries and modules.
Spark integrates with various data sources (CSV, HDFS, remote databases, etc...), actions can then be performed against the
data.

In the background, Spark achieves performance by spreading tasks across your cluster - This will sound familiar if you've ever worked with
Hadoop.

With the above in mind, I'll try to accomplish a few things with this guide:

* Download Spark
* Start a master server
* Start slave workers
* Launch an example application on the cluster

### Downloading Spark

The latest version of Spark can be downloaded [here](https://spark.apache.org/downloads.html).
Even if you're not planning on using Hadoop, a pre-built version is more than fine.

```bash
wget http://apache.mirror.anlx.net/spark/spark-1.6.1/spark-1.6.1-bin-hadoop2.6.tgz
tar -xvzf spark-1.6.1-bin-hadoop2.6.tgz
rm -rf spark-1.6.1-bin-hadoop2.6.tgz
mv spark-1.6.1-bin-hadoop2.6/ spark 
```

You should download Spark into all the nodes you want your applications to run on.

If you want to see Spark in action, run one of the example scripts. Spark comes with plenty examples for both Scala and Python 
(these can be found under examples/src/main/).

```bash
cd spark/
./bin/run-example SparkPi 10
```

Spark will do its thing and eventually return you a result.

### Starting the master server

One of your nodes needs to run as master, which will delegate tasks across the workers.

```bash
./sbin/start-master.sh
```

A master web ui should now be available at [http://localhost:8080](http://localhost:8080), it will list your
hardware and slave resources, as well as details about the applications you run against the cluster. You should also see a **Master URL**,
this will be needed to start the slave workers.

### Starting the slave workers

Slaves should be started on all nodes, including the one where the master is setup.

```bash
./sbin/start-slave.sh <Spark Master URL>
```

If you visit the Spark Master web ui, your workers should now be visible.

### Launching an example application on the cluster

We use the spark-submit script to launch applications.

```bash
./bin/spark-submit --master <Spark Master URL> examples/src/main/python/pi.py 100
```

The _--master_ option is used to specify the master URL.

You should now see your tasks being spread across your workers, Spark does quite a bit of logging by default:

```bash
16/05/03 19:50:46 INFO TaskSetManager: Finished task 85.0 in stage 0.0 (TID 85) in 180 ms...
16/05/03 19:50:46 INFO TaskSetManager: Starting task 94.0 in stage 0.0 (TID 94...
```

_And that's it!_ Spark is now ready to run as a Standalone Cluster.

If you'd like to read more on Standalone Mode click [here](https://spark.apache.org/docs/1.6.1/spark-standalone.html),
at the time of writing the latest Spark version is **1.6.1**.