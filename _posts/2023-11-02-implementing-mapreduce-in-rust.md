---
layout: post
title: "Implementing MapReduce in Rust"
---

There's been many fascinating developments in what is called "big data" recently.
Still, much of it traces back to Google's revered MapReduce, a framework for executing single instructions on large amounts of data.
So I wanted to build a simple implementation to get a sense of how all the pieces fit together.
At the same time, Rust struct my radar as a promising language with its speed, memory safety, and a decent async model.

Here's some things I learned while building MapReduce in Rust, challenges of building 
even simple distributed systems, and ideas I think are worth further pursuing.
You can find the full code [here](https://github.com/jamesma100/mrlite) - try it out if you want, but I assume no responsibility!


### Overview
MapReduce is nothing more than an abstraction for parallel computation.
You define two functions, `map()` and `reduce()`, and the framework will call those functions on your potentially large
dataset, handling computation across a cluster of machines without you having to worry about synchronization.

The user-defined `map()` function transforms your data into key/value pairs, while the `reduce()` function takes a large 
number of these key/value pairs and "reduces" them to a smaller set.

For instance, imagine a dataset that consists of customers and their spending in a day.
A `map()` function could take all the (customer name, amount) pairs and capitalize the names of the customers.
A `reduce()` function could then collect all the pairs with the same name and reduce them to just one pair, summing
up all the individual amounts.

```
// Figure 1: A simple MapReduce job

  maria  | $12.02              maria   | $41.4
  george | $14.19              cecilia | $1441.1
  maria  | $1.00               marcus  | $100.01
         |                             |
        map                           map
         |                             |
"Maria"  | $12.02            "Maria"   |  $41.4
"George" | $14.19            "Cecilia" |  $1441.1
"Maria"  | $1.00             "Marcus"  |  $100.01
         |                             |
         ----------- collect -----------
                        |
              "Cecilia" |  $1441.10
              "George"  |  $14.19 
              "Marcus"  |  $100.01
              "Maria"   |  $1.00
              "Maria"   |  $41.4
              "Maria"   |  $12.02
                        |
                     reduce
                        |
              "Cecilia" |  $1441.10
              "Marcus"  |  $100.01
              "George"  |  $14.19 
              "Maria"   |  $54.42 
``` 

Note that prior to the reduce phase, the output from the map phase is sorted by an intermediary, called here the "collect" phase.
This is essentially a preprocessing phase to so the reduce can run fast.
In Rust, these two functions could look something like:
```
use std::collections::HashMap;

fn map(pair: (&str, i32)) -> (String, i32) {
    let mut word = pair.0.chars();
    match word.next() {
        None => (String::new(), pair.1),
        Some(c) => (c.to_uppercase().chain(word).collect(), pair.1),
    }
}

fn reduce(pairs: Vec<(&str, u32)>) -> HashMap<&str, u32> {
    let mut reduced_pairs = HashMap::new();
    for pair in pairs.iter() {
        match reduced_pairs.get(pair.0) {
            None => reduced_pairs.insert(pair.0, pair.1),
            Some(val) => reduced_pairs.insert(pair.0, val + pair.1),
        };
    }    
    reduced_pairs
}
```

### Design
A basic design consists of one master node, and one or more worker nodes.
The master initializes a number of map and reduce tasks, typically specified by the user, and keeps metadata about those tasks, such as its completion status and whether it is a map or reduce task.

> Note that this distinction between map and reduce-style operations is actually important for persistence. Frameworks like Spark rely on lineage for disaster recovery - that is, logging a sequence of operations on a large number of columns rather than recording every individual, granular update.
During a failure, a map task that operates on some partition of data can be easily re-executed for just that partition. However, in the case of a reduce task, which takes on many dependencies, the system must replay not just the task itself, but all its dependee tasks.

Once a worker boots, it will send a RPC to the master, asking for a task.
The master will look through its list of available tasks and assign one to the worker.
Note that the master can only assign map tasks at first, and cannot begin assigning reduce tasks until every map task is complete.
When a worker completes a map task, it will notify the master, then becomes idle again, during which a master can assign it another task.

Once all map tasks are complete and the reduce phase begins, workers are given multiple files stored on disk, which are also the output files of the previous map phase. 
So a reduce worker must be given the location of these files, read them, call its reduce function, then write the final output onto disk.

The master must also keep track of how much time a worker is spending on a given task; if some limit is crossed, perhaps due to the worker clogging up or dying, that task is revoked from the worker and handed out to another worker.

### Implementation notes
Both the master and worker processes essentially sit in an infinite while loop, in which the former waits for a request and the latter sends one whenever it becomes idle.

For communication, we use the `tonic` gRPC framework to generate request/response types from protobuf structs, which also handles serialization for us.
Furthermore, as Rust does not have a built-in async runtime, we defer to the popular crate `tokio`.
Putting it all together, we can easily write server/client code that represents a master/worker interaction.

We also need to keep track of some shared state between the master and the workers, such as whether the entire job is finished or not.
Keeping track of this information in memory as fields in the master seemed difficult in the face of concurrent updates as well as Rust's lifetime and borrowing rules.
Plus, what if the master dies and we have to start all over again because we lose track of such basic information, despite having existing work persisted to disk?

So here we defer to use some external storage, in our case a persistent Mongo instance.
Though this only works since we are on one machine; if that were not the case, we would likely need something like Zookeeper, which is built for maintaining this sort of metadata while offering synchronization across many nodes.

> Note that in terms of maintaining MapReduce metadata, we might want to choose the strongest consistency guarantees, as we don't have a ton of client reads or writes, and the overhead of maintaining consistency across various nodes should be dwarfed by the actual processing times of the tasks.
Zookeeper, for instance, is NOT strongly consistent, but only so from a _client_ standpoint, meaning a client will not see older data than it previously read, but can still see older data than another client.
Zookeeper does this to improve read throughput, as client reads can be served from any node without having to go through the leader.
However, Zookeeper does have a `sync()` function, which ensures every client sees the latest writes, i.e. a write must be copied to every node before a read request can be served.


### Areas of work
Some areas for potential improvements, in no particular order.
- Eliminating disk I/Os!
- Better concurrency model for shared state - something like Zookeeper. Or perhaps, sharing state is the wrong answer, and we should be using message passing instead, something that Golang offers natively.
- Persistence from master crash - maybe involving a worker taking over as the master through some election process.
- Better persistence from worker crash: instead of having another worker start over a failed task, we could have some sort of checkpointing so that previous progress from a failed worker is not wasted.

### References
- MIT's [distributed systems course](http://nil.csail.mit.edu/6.824/2020/labs/lab-mr.html), where I got inspiration for the design
- Original MapReduce [paper](http://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf)
- [Spark paper](https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final138.pdf)
- Zookeeper docs on [consistency guarantees](https://zookeeper.apache.org/doc/current/zookeeperInternals.html#sc_consistency)
- [Article](https://without.boats/blog/why-async-rust/) on the background of async Rust
