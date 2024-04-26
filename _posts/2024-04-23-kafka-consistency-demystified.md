---
layout: post
title: "Kafka consistency demystified"
---

Every distributed system faces the same problem known as consistency.
That is, given a large number of machines, how do we ensure they all eventually contain the same data in spite of events like server crashes and network outages?

The first thing to consider is how to respond to a client write request that needs to be replicated.
Suppose we have a system consisting of a leader and three followers, and a client wants to write to it.
When the request reaches the leader, the simplest thing we can do is simply return a success to the client and proceed with replication asynchronously, giving us the absolute minimum response time.

However, if the leader dies at this instant, and the write hasn't propgated to the followers yet, we've lost data.

```
      1. client sends request      
      ------>                     3. leader starts replication but dies
client        leader  -------------------
      <------                           |--> follower1
    2. leader sends response            |--> follower2
                                        |--> follower3
```

On the other extreme, we can ensure that the leader replicates the write to all three followers before committing the message and responding to the client.
This will ensure maximum consistency; the write is persisted across many nodes, and we are fault tolerant.
This however, means extreme latency for the client request, which now grows linearly with respect to the number of replicas.

Thus, you can picture consistency and latency as tradeoffs on a Pareto frontier, where it is impossible to improve one without sacrificing the other.
How we reach any point on the curve is beyond the coverage of this post, but could include things like code correctness and sufficient hardware capacity.

In a practical scenario, we would want some sort of middle ground - replicate every client write to _some_ number of replicas `n`, where `n < m`, and `m` being the total number of replicas.

How Kafka does distributed consensus is rather interesting, so I wanted to take the time to discuss its approach and how it differs from more traditional models like Raft.

### Leader re-election
Let's begin by examining a common problem - leader election.
If the leader dies, we need to choose a new leader to take its place from a pool of qualified candidates.
Ideally, we would like that pool to be large so we find a qualified candidate faster.

How do we come up with this candidate pool?
Since we want the new leader to have all written data we've acknowledged to our clients, we need to make sure the new leader is _caught up_ with the previous leader.

Note that the number of candidates we have to go through depends on the number of _follower acknowledgments_ - that is, the number of followers we replicate each request to before returning a response to the client.

If we have 10 servers and we require 9 to acknowledge each write, in the worst case we only need to check 2 candidates in the event of a leader crash (assuming the the first one we pick is the single follower _not_ caught up).
On the flip side, if we only require 3 up-to-date replicas, we would have to go through at most 8 candidates before finding a qualified leader.
In general, given `n` replicas, the number of candidates to evaluate in the face of a leader crash is `n - w + 1`, where `w` is the number of follower acknowledgements.

A common approach is to require a _majority_ of replicas to be in sync with the leader; that is,  given `2n + 1` nodes, we require `n + 1` acknowledgements.
Now, if the leader crashes and we go through a candidate pool of `n + 1` followers to choose a new leader, we are guaranteed to find one that is up to date since there will necessarily be an overlap.

### Kafka and ISRs
Kafka does not do a majority-based quorum, but rather hard-selects a list of what it calls ISRs (in-sync replicas).
Every write request requires that it is replicated to all ISRs to be committed, and the number of ISRs is configurable via a user property `min.insync.replicas`.

```
min.insync.replicas = 3
ISRs: leader, follower1, follower2

        |--------------------------------------|
server: |  leader      follower1    follower2  |  follower3 (lagging!)
log:    | [1, 2, 3]    [1, 2, 3]     [1, 2, 3] |   [1, 2]
        |--------------------------------------|
                        ISRs
```
In the example above, it's okay for follower 3 to lag, since it's not in the ISR.

The reason for this approach is that for majority-based systems, the _ratio_ between in-sync replicas to total replicas is too small.
Namely, to have `n + 1` in-sync replicas, we need at least `2n` _total_ replicas.
Usually, we can't afford to have that many total replicas.
But if we reduce the number of total replicas, the number of in-sync replicas becomes unacceptably low.
Using ISRs allows every system to dynamically navigate this tradeoff to fit its own requirements.

### What if I choose a majority for my ISRs?
Suppose we have `2n` replicas.
How would choosing `n + 1` as our number of ISRs differ from a majority quorum approach, which would also require `n + 1` replicas to be in sync?

The difference here is that in the ISR case, we are selecting `n + 1` _specific_ followers that need to be up-to-date with the leader.
On the other hand, the majority quorum case simply requires _any_ `n + 1` followers to acknowledge writes.

The practical difference between the two scenarios is that the latter has a significant latency advantage.

Suppose we have 10 servers numbered 1 to 10 and ordered by latency (server 1 is the fastest with a 1s tail latency), and we choose our `min.insync.replicas = 6`, a majority.
In the best case scenario, our ISR set include servers 1 to 6, so the latency of any client request is 6s.
In the worst case scenario, our ISR is servers 5 to 10, giving us a 10s latency.

On the other hand, if we use a majority quorum, all 9 followers receive the replication request from the leader, and servers 2 to 6 complete their replication and send their acknowledgements first, so our latency is always 6s.

In other words, using ISRs causes latency unpredictability as it depends on which specific ISRs are chosen and their response time, while using a majority quorum caps our latency by the fastest servers.

### Client reads and split brains
Given that the leader always contains the most up-to-date date, can we be guaranteed that if as long as we route reads through the leader, clients will always get the latest data?
The answer is no.

Suppose a leader gets cut off from the rest of the cluster, which goes on to elect a new leader.
The now deposed leader no longer gets any writes, since clients now write to the new leader, so will return stale data to the client.
To prevent this scenario from happening, any time a leader receives a client request, it must contact the rest of the cluster, usually a majority, and get confirmation that it is indeed the rightful leader before responding to the client.
Doing this, however, will make latency depend on the size of our cluster.

Recall that Kafka maintains a set of ISRs that are guaranteed to be up-to-date.
So if a client wants the latest data, we just need to make sure that client reads from an ISR.
But since ISRs are maintained by the leader, we still face the issue of leader deposition.
In other words, how do we check that the leader is indeed the leader, as agreed upon by the rest of the cluster?

Contacting a majority of the cluster like we mentioned earlier would be too costly, so Kafka optimizes this using Zookeeper and a select broker called the controller, which is responsible for communicating with Zookeeper, observing the state of all brokers, and electing new leaders.

So leader verification is a simply a matter of maintaining an active connection with Zookeeper, which the leader accomplishes by sending periodic heartbeats to our Zookeeper cluster. 

But this doesn't solve the issue of stale reads.
Since Kafka actually allows reads to be served by any replica, not just the leader, leader verification does not imply up-to-date reads.
You would need a large `min.insync.replicas` to reduce the likelihood of a stale read.

But even then, consider the window after the leader disconnects but before a missing heartbeat is detected.
In that case, the leader could continue serving writes until the controller notices and proceeds to elect a new leader.
The data sent to the old leader would then be lost.
Even after it reconnects, the new leader would force it to overwrite its data to replicate its own (more on this later).

The key here is that a split brain can still occur despite ISRs and to choose the heartbeat window `zookeeper.session.timeout.ms` to be small such that such an event is unlikely to occur.

### Handling inconsistencies
When a leader crashes and a new leader is selected, there will often be inconsistencies between the new leader, the old leader, and the followers.
Kafka prevents this by only choosing leaders from the ISR set, which is guaranteed to contain the latest committed entries.

An edge case occurs when there is one ISR left - the leader.
Imagine the scenario with servers S0, S1 and S2 and with ISR = (S0, S1, S2).
 1. S0 is initially a leader, successfully replicates some entry `e7` to S1 and S2, satisfying the ISR invariant
 2. Entry `e7` is committed - S1 and S2 successfully flush it to disk.
 3. S1 and S2 crash, and are both removed from the ISR
 4. S0 crashes. Kafka does not let ISR drop to zero, so S0 remains a ISR.
 5. All three servers S0, S1, S2 recover.
 6. S0 becomes leader, since it is the only one in ISR
 7. Note that S0 never flushed `e7`, but since it became leader, it is now the source of truth, and force S1 and S2 to adopt its log, meaning `e7` is permanently lost.

[KIP-966](https://cwiki.apache.org/confluence/display/KAFKA/KIP-966%3A+Eligible+Leader+Replicas) resolved this issue.

### A whirlwind tour of persistence
You may be wondering why it's possible that at step 2, `e7` was committed but not persisted to all the servers.
Doesn't "committed" mean that the entry is stored on every ISR, or a majority of servers in the majority quorum case?

The answer lies in how operating systems handle writes and how exactly we define persistence.

Consider what happens when a server receives a request to write an entry to its log.
- The server receives the request and stores the entry in memory
- The server calls the `write()` syscall to send it to disk.
- The OS context-switches to kernel mode, as `write()` is a privileged operation, and copies the entry into a kernel buffer.
- After a while, the OS calls `fsync()`, copying the entry in the kernel buffer to the disk controller.

Kafka does not require `fsync()` to be called on every write, so after step 3, when we call `write()`, the data is considered persisted.
If the OS crashes after we `write()` but before we `fsync()`, we lose all our buffered data.
In the case of Kafka, this would mean triggering a rollback procedure to recover lost data.

To be safer, we could instead define persistence as some point in time _after_ `fsync()` is called.
This would obviously slow things down, as we can no longer buffer data in the kernel but would allow us to _increase_ our persistence.

Note that I used the word "increase" and "safer" to describe persistence, and never "safe" or "guarantee."
This is because persistence is never absolute.
Even getting a successful return from `fsync()` doesn't guarantee persistence, as your disk likely stores data in its track cache until enough data has accumulated, rather than potentially waiting for a full revolution of a disk platter on every read.

We could even go one step further and assume data is physically on storage media (magnetic tape, NAND chip, etc.), but your data center burns down and all your disks are destroyed.
This is a rather absurd hypothetical, but the takeaway here is that the meaning of persistence depends on context and interpretation.

Going back to Kafka, we can see that the last replica standing issue can only occur because Kafka doesn't flush on every write, but rather relies on rollback and recovery after a crash.

### Uncommitted inconsistencies
Note that so far we've only discussed inconsistencies of _committed_ data - uncommitted data can still be inconsistent.
We saw this earlier with the split brain scenario, where a deposed leader still thinks its the leader and thus continues to serve writes, only to be overwritten when it rejoins the cluster.
However, we don't consider this an inconsistency, despite the follower having data that the new leader does not.
Since it hasn't been committed, it is only a problem from our _cluster's_ perspective; to the client, the data simply does not exist.

In other words, this is an internal problem we've abstracted away from the client.

The simplest way to handle this is to force all followers to adopt the leader's log, which is exactly what Kafka does.

It's interesting to see how this is done.
In Raft for instance, if a newly elected leader discovers an inconsistency between its own log and a follower's log, it moves backwards in its log, sending RPCs until it finds the latest point of agreement.
Then, it moves forward and overwrites each inconsistent entry in the follower's log with its own.

```
leader:   [1, 2, 3, 4, 5, 6]
follower: [1, 2, 1, 0, 9, 7]
index:     0  1  2  3  4  5

time                           follower log
0                         |   [1, 2, 1, 0, 9, 7]
1                      |      [1, 2, 1, 0, 9, 7]
2                   |         [1, 2, 1, 0, 9, 7]
3                |            [1, 2, 1, 0, 9, 7]
4             |               [1, 2, 1, 0, 9, 7] * found agreement!
5                |            [1, 2, 3, 0, 9, 7]
6                   |         [1, 2, 3, 4, 9, 7]
7                      |      [1, 2, 3, 4, 5, 7]
8                         |   [1, 2, 3, 4, 5, 6] * now consistent w/ leader
```
In the above example, the leader starts from the latest entry at index 5, moving backwards in the log until it finds the latest common entry with the follower, namely the entry at index 1.
It then moves forward and replaces entries 2 to 5 in the follower's log with its own entries.

### Closing thoughts
This post is already getting longer than I would like, and I think I've shed enough light on the many ways distributed systems can go awry and the tricks Kafka employ to deal with them.
Though it's unlikely you'll ever need to worry about any of this, I think it's always interesting to understand a bit about what's going on behind the gates of abstraction.
