---
title: "Rate-limiting in Go"
date: 2017-02-24
keywords:
- go
- golang
- rate limiting
- go rate limiting
- exponential backoff
- algorithm
---

A couple of weeks ago I wrote a post on [Data Pipelines with Go, Kafka and Cassandra](/data-pipelines-cassandra-kafka-and-python-and-go.html). I feel like this is a good opportunity to write about rate limiting.

Why would you need rate limiting in your data pipeline? Realistically, heavy data streams can't always be processed
at the most optimal speed. There are constant limitations: Hardware; Network; Data spikes, etc.

In this specific case, I am going to address Cassandra timeouts, and how rate limiting can come to the rescue.

The solution is broken down into 2 areas.

### 1. Limiting the amount of concurrent goroutines

This is a good idea on itself. Consuming each Kafka message on its own goroutine, without any limit, is a good way to kill your Cassandra cluster really quickly (especially in budget hardware). Maybe your current ingestion rate doesn't justify it, but what if the data rate increases? Maybe you have a spike in traffic, or you need to batch a large data set. Whatever the reason, you should always plan ahead.

If Cassandra times out on a write (the cluster is under heavy load), we want to keep that goroutine "ransom" and prevent others from starting... Which brings us to:

### 2. Leveraging the Exponential Backoff algorithm

On a timeout, we want to keep retrying the write until we succeed. To achieve this we make use of the [Exponential Backoff](https://en.wikipedia.org/wiki/Exponential_backoff) algorithm:

> Exponential backoff is an algorithm that uses feedback to multiplicatively decrease the rate of some process, in order to gradually find an acceptable rate.

Essentially, we slow down the attempts to write to Cassandra as soon as we hit a timeout. Luckily,  someone in the Go community has already done the [hard work](https://github.com/cenkalti/backoff) for us by implementing the algorithm in an easy to use package.

## The code

We modify our Consumer loop to limit the amount of concurrency:

```go
lim := make(chan bool, < limit >)

// Consume!
for {
    select {
    case msg := <-consumer.Messages():
        consumer.MarkOffset(msg, "")
    
        // Don't start new goroutines if the channel is "full".
        lim <- true
    
        // We now need to pass the lim channel to our goroutines, as well.
        go Process(msg, cassSession, lim)
    case err := <-consumer.Errors():
    log.Println("Failed to consume message: ", err)
    }
}
```

I recommend a dynamic value for **< limit >**, implementation is entirely up to you, ex: Check for overall cluster performance on small intervals and dynamically update this value; Monitor your topic lag and increase/decrease this value.

Now we modify the Process function:

```go
func Process(msg *sarama.ConsumerMessage, session *gocql.Session, lim chan bool) error {
    defer func() {
        // Free the channel.
        <-lim
    }()

    // msg validation goes here...
    
    // Configure the Exponential Backoff Package.
    b := backoff.NewExponentialBackOff()
    b.InitialInterval = 1 * time.Millisecond

    op := func() error {
        // Output the elapsed time between query attempts.
        fmt.Println(b.GetElapsedTime())
        
        // Attempt Insert.
        err := s.Query(q, v...).Exec()
        
        return err

    }
    
    // Retry until err = nil.
    _ = backoff.Retry(op, b)
    
    // Any other checks could go in here...
    
    return nil
}
```

First we added a defer function, to ensure the channel is freed on return. Secondly, we attempt to write to Cassandra (you probably want to add some logic to specifically check for timeout errors, as well as log/report on errors). If the Query.Exec function returns an error, the execution will hold and retry at a later point in time.

As an example, if a query kept timing out under the above setup, this would be a possible output:

```terminal
111ns
1.551808ms
3.930745ms
7.356792ms
11.490968ms
17.408666ms
27.088205ms
33.72182ms
45.188354ms
62.221412ms
93.112465ms
155.354895ms
271.031533ms
365.777994ms
542.306919ms
782.937858ms
1.208119765s
```

Since we are progressively slowing down the attempts to write to Cassandra, and while doing so, not freeing up the channel - **We accomplish rate-limiting.**