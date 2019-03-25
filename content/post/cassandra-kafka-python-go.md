---
title: "Data Pipelines: Cassandra, Kafka and Python (and Go!)"
date: 2017-02-06
keywords:
- go
- golang
- kafka
- cassandra
- big data
- data pipeline,
- gocql
- sarama
---

Last year I started working on a 'Big Data' exercise. It's an ongoing project
that mixes large amounts of web traffic, data ingestion and analytics. It's also really fun. We get to
play with an array of new technologies - sometimes on a bet, granted - but most of the time it pays off.

One of the early requirements was to build a 3 stage "Data Pipeline":

1. Receive data. ([Kafka](https://kafka.apache.org/))
2. Validate and manipulate data. (Python)
3. Store data. ([Cassandra](https://cassandra.apache.org/))

I won't justify the use of Kafka and Cassandra (that might be a topic for another post), both fit our needs and perform as advertised. I will talk about Python, though.

### Python is for everything!... Or is it?

Python is a fantastic language. We use it extensively for our web and analytics needs.
When we saw tools like [PyKafka](https://github.com/Parsely/pykafka) and the
[DataStax Python Driver for Apache Cassandra](https://github.com/datastax/python-driver), it felt natural to continue
using it to write our Data Pipeline.

For our current needs, Python handles the job... _Most of the time_.
There are days when we need to batch large sets of data, or our traffic hits abnormally high levels for a short period of time --
And that's when our Kafka topics start to grow.

To combat this we create more PyKafka consumers, dozens and dozens of Python processes working away trying to bring that topic lag down... But it's
no use, we always hit one of two limits: consumers or hardware.

We know Kafka is blazing fast. Cassandra is also eager to show us
[how fast](http://techblog.netflix.com/2014/07/revisiting-1-million-writes-per-second.html) it can write. On the other hand, our Python
"middle man" is slow. We leverage the language and the libraries to the best of our abilities, but it's clear we are losing the write/read war.

**Note:** This isn't criticism towards Python. We knew we were taking a risk by not going down the compiled
road for this sort of job. Still, I'm sure there are folks out there who can truly milk performance out
of Python for these sort of data ingestion tasks - I'd love to hear success stories!

### Enter Go

I have recently started programming in Go. Coming from an academic C background (but lacking the professional experience), it felt refreshing. I really like the
standard library, the consistency and "cleanliness" of the language, the [excellent](https://golang.org/doc/) documentation.

As I entered the topic of concurrency (goroutines, channels, selects, etc...) I couldn't help but see the potential for our Data Pipeline.
I thought it would be worth testing Go with a simplified version of our current process:

* Consume messages from Kafka.
* Parse the JSON-encoded message and validate the data.
* Insert data into Cassandra tables.

The rest of the post will cover the above technical details.

### The Setup

First we are going to need a Cassandra table to write data to:

```sql
CREATE TABLE go_test.site_stats (
    date date,
    site_id int,
    views counter,
    clicks counter,
    PRIMARY KEY ((date), site_id)
);
```

The data being written to Kafka are simple JSON-encoded messages with the following structure:

```json
{
    "date": "2017-02-04",
    "site_id": 3491,
    "views": 10,
    "clicks": 6
}
```

### The code

It would be unfair not to mention the packages that make all of this possible:

* [Sarama](https://github.com/Shopify/sarama), the Kafka client.
* [Sarama Cluster](https://github.com/bsm/sarama-cluster), cluster extensions for Sarama.
* [gocql](https://github.com/gocql/gocql), the Cassandra client.

I also make use of [ozzo-validation](https://github.com/go-ozzo/ozzo-validation) -
My preferred data validation package.

The code below is an oversimplification of our own process, but it should be a good starting point
if you're new to Go or have the desire to learn about Kafka/Cassandra.

First we will define the update query, the update structure and validation function:

```go
const (
    // SiteQuery is the CQL update query we are running.
    SiteQuery string = `UPDATE site_stats SET
                views = views + ?,
                clicks = clicks + ?
                WHERE date = ?
                AND site_id = ?`
)

// Update is the structure expected from a Kafka message.
type Update struct {
    Date   string `json:"date"`
    SiteID int    `json:"site_id"`
    Views  int    `json:"views"`
    Clicks int    `json:"clicks"`
}

// ValidateData ensures the message data is ready for ingestion.
func (u Update) ValidateData() error {
    return validation.StructRules{}.
        Add("Date", validation.Required, 
            validation.Match(regexp.MustCompile(`\A\d{4}(-\d{2}){2}$`)).Error("Invalid datetime supplied")).
        Add("SiteID", validation.NotNil).
        Add("Views", validation.NotNil).
        Add("Clicks", validation.NotNil).
        Validate(u)
}
```

We can now move into our main() function and start by establishing a connection to Cassandra:

```go
cassCluster := gocql.NewCluster("< Comma separated list of IPs >")
cassCluster.Keyspace = "go_test"
cassSession, err := cassCluster.CreateSession()

if err != nil {
    log.Fatal(err)
}
defer cassSession.Close()
```

We do the same thing for Kafka, as well as basic configuration:

```go
kafConfig := sarama_cluster.NewConfig()
kafConfig.ClientID = "go_test"
kafConfig.Consumer.Return.Errors = true
kafConfig.Consumer.Offsets.Initial = sarama.OffsetOldest

addrs := []string{"<ip>:<port>"}
kafClient, err := sarama_cluster.NewClient(addrs, kafConfig)

if err != nil {
    log.Fatal(err)
}

// Validate Config.
err = kafConfig.Validate()

if err != nil {
    log.Fatal(err)
}
```

Now it gets interesting! We get our consumer up and running:

```go
// Create consumer.
var topics = []string{"go_test"}
consumer, err := sarama_cluster.NewConsumerFromClient(kafClient, "go_test", topics)

if err != nil {
    log.Fatal(err)
}
defer consumer.Close()

// Consume!
for {
    select {
    case msg := <-consumer.Messages():
        consumer.MarkOffset(msg, "")
        go Process(msg, cassSession)
    case err := <-consumer.Errors():
        log.Println("Failed to consume message: ", err)
    }
}
```

_How simple is that?_ Notice that when a message is consumed, a **Process()** function is invoked in a goroutine.
Here is that function:

```go
// Process validates data and writes to Cassandra.
func Process(msg *sarama.ConsumerMessage, session *gocql.Session) {
    u := Update{}
    // Unmarshal against the struct type.
    err := json.Unmarshal(msg.Value[:], &u)

    if err != nil {
        log.Fatal(err)
    }

    // Validate the data.
    err = u.ValidateData()

    if err != nil {
        log.Fatal(err)
    }

    /*
    Extra validation/operations could go in here...
    */

    var args []interface{}
    args = append(args,
        u.Views,
        u.Clicks,
        u.Date,
        u.SiteID)

    err = session.Query(SiteQuery, args...).Exec()

    if err != nil {
        log.Fatal(err)
    }
}
```

Each message is Unmarshal'd, validated and _finally_ inserted into Cassandra.

**Note:** If you want to execute more than one query (and depending on your use case) you might want to look
into Batching. [This](https://christopher-batey.blogspot.co.uk/2015/02/cassandra-anti-pattern-misuse-of.html) 
is a great read on the subject, it points out exactly when you might benefit from it. Anyway, gosql supports
the feature.

_And we are done!_ Here is the full code:

```go
package main

import (
    "encoding/json"
    "log"
    "regexp"

    "github.com/Shopify/sarama"
    sarama_cluster "github.com/bsm/sarama-cluster"
    validation "github.com/go-ozzo/ozzo-validation"
    "github.com/gocql/gocql"
)

const (
    // SiteQuery is the CQL update query we are running.
    SiteQuery string = `UPDATE site_stats SET
                views = views + ?,
                clicks = clicks + ?
                WHERE date = ?
                AND site_id = ?`
)

// Update is the structure expected from a Kafka message.
type Update struct {
    Date   string `json:"date"`
    SiteID int    `json:"site_id"`
    Views  int    `json:"views"`
    Clicks int    `json:"clicks"`
}

// ValidateData ensures the message data is ready for ingestion.
func (u Update) ValidateData() error {
    return validation.StructRules{}.
        Add("Date", validation.Required,
            validation.Match(regexp.MustCompile(`\A\d{4}(-\d{2}){2}$`)).Error("Invalid datetime supplied")).
        Add("SiteID", validation.NotNil).
        Add("Views", validation.NotNil).
        Add("Clicks", validation.NotNil).
        Validate(u)
}

func main() {
    /*
    CASSANDRA CONFIG
    */
    cassCluster := gocql.NewCluster("< Comma separated list of IPs >")
    cassCluster.Keyspace = "go_test"
    cassSession, err := cassCluster.CreateSession()

    if err != nil {
        log.Fatal(err)
    }
    defer cassSession.Close()

    /*
    KAFKA CONFIG
    */
    kafConfig := sarama_cluster.NewConfig()
    kafConfig.ClientID = "go_test"
    kafConfig.Consumer.Return.Errors = true
    kafConfig.Consumer.Offsets.Initial = sarama.OffsetOldest

    addrs := []string{"<ip>:<port>"}
    kafClient, err := sarama_cluster.NewClient(addrs, kafConfig)

    if err != nil {
        log.Fatal(err)
    }

    // Validate Config.
    err = kafConfig.Validate()

    if err != nil {
        log.Fatal(err)
    }

    // Create consumer.
    var topics = []string{"go_test"}
    consumer, err := sarama_cluster.NewConsumerFromClient(kafClient, "go_test", topics)

    if err != nil {
        log.Fatal(err)
    }
    defer consumer.Close()

    // Consume!
    for {
        select {
        case msg := <-consumer.Messages():
            consumer.MarkOffset(msg, "")
            go Process(msg, cassSession)
        case err := <-consumer.Errors():
            log.Println("Failed to consume message: ", err)
        }
    }
}

// Process validates data and writes to Cassandra.
func Process(msg *sarama.ConsumerMessage, session *gocql.Session) {
    u := Update{}
    // Unmarshal against the struct type.
    err := json.Unmarshal(msg.Value[:], &u)

    if err != nil {
        log.Fatal(err)
    }

    // Validate the data.
    err = u.ValidateData()

    if err != nil {
        log.Fatal(err)
    }

    /*
    Extra validation/operations could go in here...
    */

    var args []interface{}
    args = append(args,
        u.Views,
        u.Clicks,
        u.Date,
        u.SiteID)

    err = session.Query(SiteQuery, args...).Exec()

    if err != nil {
        log.Fatal(err)
    }
}
```
