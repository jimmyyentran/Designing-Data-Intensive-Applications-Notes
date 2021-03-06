<!-- TOC -->

- [4. Encoding and Evolution](#4-encoding-and-evolution)
  - [Dataflow through databases](#dataflow-through-databases)
  - [Dataflow Through Services: REST and RPC](#dataflow-through-services-rest-and-rpc)
    - [Web services](#web-services)
    - [Problem with RPCs](#problem-with-rpcs)
      - [Current directions for RPC](#current-directions-for-rpc)
    - [Data encoding and evolution for RPC](#data-encoding-and-evolution-for-rpc)
  - [Message-Passing Dataflow](#message-passing-dataflow)
    - [Message broker](#message-broker)
      - [Distributed actor frameworks](#distributed-actor-frameworks)
  - [Summary](#summary)
- [5. Replication](#5-replication)
  - [Part II. Distributed Data](#part-ii-distributed-data)
    - [Replication vs Partitioning](#replication-vs-partitioning)
  - [5. Replication intro](#5-replication-intro)
    - [Leaders and Followers](#leaders-and-followers)
      - [Synch vs Async Replication](#synch-vs-async-replication)
      - [Setting up new Followers](#setting-up-new-followers)
      - [Handling node outages](#handling-node-outages)
      - [Implementation of Replication Logs](#implementation-of-replication-logs)
    - [Problems with Replication Lag](#problems-with-replication-lag)
      - [Reading your own writes](#reading-your-own-writes)
      - [Monotonic Reads](#monotonic-reads)
      - [Consistent Prefix Reads](#consistent-prefix-reads)
      - [Solutions for Replication Lag](#solutions-for-replication-lag)
    - [Multi-Leader Replication](#multi-leader-replication)
      - [Use cases for MLR](#use-cases-for-mlr)
      - [Handling Write Conflicts](#handling-write-conflicts)
      - [Multi-Leader Replication Topologies](#multi-leader-replication-topologies)
    - [Leaderless Replication](#leaderless-replication)
      - [Writing to DB when node is down](#writing-to-db-when-node-is-down)
      - [Limitations of Quorum Consistency](#limitations-of-quorum-consistency)
      - [Sloppy Quorums and Hinted Handoff](#sloppy-quorums-and-hinted-handoff)
      - [Detecitng Concurrent Writes](#detecitng-concurrent-writes)
    - [Summary](#summary-1)

<!-- /TOC -->

# 4. Encoding and Evolution

## Dataflow through databases

- When older version of the application updates data previously written
  by a newer version of the application data may be lost

![5326b302.png](attachments/5326b302.png)

## Dataflow Through Services: REST and RPC

- Clients and servers
  - XMLHttpReqeust inside client-side JS so not HTML but something
    understandable by the client side
- **Service-oriented architecture (SOA)**: Decompose large application
  to smaller services by area of functionality
  - AKA **Microservices architecture**
- Services are similar to DB in sense that they allow submit and query
  data in specified format
- Key for service-oriented/microservice is to make application easier to
  change by making services independently deployable and evolvable
  - Team owns service allows old and new versions running

### Web services

Whenever HTTP is used as underlying protocol But webservices are used
not only on the web but different context:

- Client app on user device
- One service to another service
- Service to different organization

**REST**: Philosophy design built around principles of HTTP

- Simple data format
- URL for identifying resources
- Cache control, autho, content type negotiation
- More popular than SOAP for microservices
- **OpenAPI or Swagger**: Way to describe API and produce doc

**SOAP**: XML-based protocol

- Most common over HTTP though aims to be HTTP independent
- Complex standards
- **WSDL**: Web Services Description Language
  - Code-generation so client can access w/ local classes and methods
  - Not human-readable so heavy use on tools
- Fallen out of favor with smaller companies

### Problem with RPCs

**RPC**: Remote procedure calls

- Hide request to network service to look same as function. AKA **Local
  transparency**
- Flawed since network calls are different than local function calls
  - local calls are predictable, network calls aren't
  - local calls return result, throw, or never return. Network calls
    also has return w/o result - timeout
  - Retry may have went through but responses are lost
    - **idempotence**: deduplication
  - Network calls are slower and latency is variable
  - Local calls pass in references to local memory. Network calls need
    to encode to bytes which are problematic for larger objects
  - Client and service can be implemented in different languages. Ex JS
    number max 2^53

#### Current directions for RPC

- It isn't going away.
- New RPC frameworks which don't hide it's a network call
  - Rest.li uses **promises** to encapsulate async calls and requrest
    multiple services in parallel
- **Service Discovery** Client find out which IP and port
- Binary encoding format = better performance than JSON over REST.
  - REST advantage is easier experimentation and debugging
  - More tools with REST

### Data encoding and evolution for RPC

- Servers update first, clients second. BC on request and FC on resp
- RPCService compat harder with cross org since no control over client
  (REST too)
- RPC no agreement on how API versioning work. Rest use version #

## Message-Passing Dataflow

**async message-passing systems**: Middle between RPC and DB.

- Similar to RPC in that req are delivered w/ low latency
- Similar to DB since message is not via direct network con but through
  message broker

Message broker adv (Kind of like SQS):

- Buffer if recipient is overlaoded
- Redeliver message if crash. Loss recovery
- Sender doesn't need to know IP and port of recipient
- One message to many recipient
- Decouples sender and client

Process is _one way_ since sender do doesn't expect resp

### Message broker

- Old uses commerial enterprise
- New: RabbitMQ, ActiveMQ, HornetQ, NATS, and Apache Kafka
- Can chain consumer to ingest topic then publish another topic

#### Distributed actor frameworks

**actor model**: programming model for concurrency in a single process
rather than dealing with threads.

- Communicate y sending async messages
- Mesasge delivery not guaranteeed
- _distributed actor frameworks_: Scale app accross multiple nodes.
  - Combine message broker and actor programming model
  - BW and FW compat to have rolling upgrades

## Summary

- encoding affects efficiency
- Rolling upgrades
  - Incremental changes for faster iteration and catch bugs
  - **evolvability**: ease of making changes to app
  - **BW compat**: new code can read old data
  - **FW compat**: old code can read new data
- encoding formats
  - Programming language: specific encoding restricted to single
    language and fail to provide FW and BW compat
  - Text: JSON, XML, CSV: Widespread, vauge of datatypes, numbers and
    binary string harder
  - Binary schema: efficeint encoding w/ FW and BW compat
    - Good for documentation
    - Code generation in statically typed
    - Now human readable
- Dataflow
  - DB
  - RPC and REST
  - Async message passing: Nodes communicate by sending messages encoded
    by sender, decode by recipient

# 5. Replication

## Part II. Distributed Data

**Scalability**: Read or write larger than single machine, need to
spread across multiple **Fault tolerance/high availability**: App
continues to work if multiple machines go down **Latency**: Servers
close to user

Shared-memory architecture and shared-disk architecture both has
downsides of cost when scaling.

**Share nothing architecture** Each machine runs independently as a node

- Distributed systems has constraints and tradeoffs
- Complexity of application and limits expressiveness

### Replication vs Partitioning

**Replication**: Keeping same data on different nodes

- Provide redundancy: if nodes are unavailable, data can still be served
  from other
- Can improve performance

**Partitioning**: Split DB to smaller subsets. **Sharding**

## 5. Replication intro

Why replicate?

- geographically close
- System continue working even if some parts fail
- Scale out number of machines that can serve read queries

Learnings:

- Difficulties in replication is handling changes to replicated data
- Single-leader, multi-leader and leaderless
- sync and async replication
- eventual consistency

### Leaders and Followers

How to ensure data ends up in all replicas

- **Leader based replication** aka active/assive or master-slave
  replication

  - 1 replica is the leader, write goes here
  - Other are followers. Leader send change to followers as part of
    replication log or change stream
  - Client need to read query from leader or followers. Write only on
    leader

  ![7f8fa9d8.png](attachments/7f8fa9d8.png)

- Relational and non relational uses this method.
- Distributed message brokers like Kafka and RabbitMQ also uses this

#### Synch vs Async Replication

- pros of sync: guarantee of up-to-date copy o fdata
- con of sync: if follower doesn't erspond, leader will block all writes
- **semi-synchronous**: only one sync. if that sync follower is down or
  snow, will elect new sync follower

Leader based replication usually is async. But if fails before writes
not replicated, can lose the write even if confirmed

- Advantage: leader can process writes even if followers behind
- useful if many followers and geo distribute

#### Setting up new Followers

Method without downtime:

1. Take a snapshot of leader point in time
2. Copy snapshot to new node
3. Follower connects to leader and request for changes since snapshot.
   Snapshot need to be associated with time
4. When follower is caught up, it process changes from leader

#### Handling node outages

**High availability**: Keep system running even single node fails

**Follower failure**: Catch-up recovery. Keep log of last transaction
processed before fault. When connect again catch up on stream of data

**Leader failure**: One leader needs to be promoted to new leader.
**Failover** clients reconfigured to consume from new leader.

- Determine leader failed: Node doesn't respond within time limit = dead
- Choose new leader: Election. Replica with most up-to-date
- Reconfigure to use new leader: Client needs to send write to new
  leader. System needs to make sure old leader becomes a follower once
  it's up again

Failover issues:

- If async replication, new leader may not receive all writes from old
  leader before old leader failed. Most common: discard old leader's
  write
- Discarding write is dangerous if other storage system outside of DB
  needs to be coordinated. Ex. Redis and MySQL storage when MySQL db has
  new inconsistent leader
- **Split brain**: two nodes think they're leader and writes become
  conflicts
- Write timeout too short = unecessary failover. Too long = longer time
  to recover

#### Implementation of Replication Logs

##### Statement

**statement-based replication**: Write requests executed sent to
followers

Possible ways to breakdown:

- nondeterministic function like NOW and RAND gets different valueon
  replicas
- Autoinc needs to be in order on each replica
- Triggers, stored procedures etc may work differently for each replica

##### Write ahead log shipping

- Append only log containin gall writes to DB
- Leader writes to log and send to followers
- Con: Logs are low level so replication is coupled with storage engine
  - If format changes, different version won't work between leader and
    followers
  - If replication protocol doesn't allow version mismatch = downtime
- Upgrade version by 1st upgrade followers, then failover to newly
  upgraded node as leader

##### Logical (row based) log replication

Different log format (**logical log**) for replication and for storage
engine which allows decoupling from storage engine.

##### Trigger-based replication

Useful if only replicate subset of data, replicate one kind of DB from
another, or resolution logic

**trigger** automatically run code when data change so can control where
and how to replicate data

More error prone to bugs and limitations than DB built in replication

### Problems with Replication Lag

Leader-based replication good for small percent write and large read

_read scaling architecture_: increase capacity of read by adding more
followers

- Works good with async
- Sync will be unreliable as single node down make system unavailable
  for writing

**eventual consistency**: Read on leader and follower is different since
writes haven't been reflected in follower

- Delay could be fraction of second or minutes if max capacity or issue
  with network

#### Reading your own writes

![cdeffbc9.png](attachments/cdeffbc9.png)

**read-after-write consistency**: User will see updated submitted
immediately

- Read user profile from the leader. Need way to know if user is looking
  at user's own profiel
- Track time of last update and recent changes pull from leader
- Client remember recent write, systems can make sure updates reflect
  until that timestamp
- Distributed across geo is harder
- Cross devices
  - Client timestamp doesn't work bc one device doesnt know other device
    last update timestamp = metadata centralized
  - Across datacenter = need to locate devices to same datacenter

#### Monotonic Reads

**moving backward in time**: anomaly with async followers
![1dd0d2ee.png](attachments/1dd0d2ee.png)

- **Monotonic reads**: no back in time reads
- User read from single replica based on hash
- If fail then need to route to different replica

#### Consistent Prefix Reads

- Inconsistent order in data read
  ![1b408b33.png](attachments/1b408b33.png)
- **Consistent Prefix Reads**: write happens in order, anyone reading
  will see them appear in order
  - Occurs when DB are partitioned or sharded. IE User A in one shard,
    user B in another
  - Solution is to make writes that are causually related written in
    same partition but cannot be efficient

#### Solutions for Replication Lag

Solutions explored above can make application code complesx

Eventuall consistency is inevitable in scalable system

### Multi-Leader Replication

- More than one node accept writes
- **multi-leader configuration**: leader acts as follower and leaders

#### Use cases for MLR

##### Multi-datacenter operation

- leader in each datacenter and they both replicate each other's changes
- Performance: writes to local leader is faster so perceived performance
  is better
- Outage tolerance: data center operate independently
- network tolerance: async replication tolerate network problems better

**Conflict resolution**: Same data concurrently modified in different
datacenters and conflicts need to be resolved

Retrofitted features in many DB can lead to quirks

##### Clients with offline operation

Moblie devices with offline capability needs to be sync is in effect
like 'leaders' with multi-leader replication process taken to the
extreme

##### Collaborative editing

- To avoid conflicts need lock on the document before editing If someone
  else wants to edit the doc, they need commit change then release
- Faster collaboration - unit of change smaller to avoid locking

#### Handling Write Conflicts

![](attachments/2020-05-16-06-23-49.png)

##### Sync vs Async conflict detection

In sync, leader just apply lock on item and abort 2nd request In async,
both writes are accepted and conflict resolved async later

##### Conflict avoidance

Route user's data to same datacenter so from user's POV it looks like
single-leader.

##### Converging towards consistent state

- Unique ID - timestamp, UUID, etc and pic the highest ID. **Last write
  wins (LWW)**. Prone to data loss
- Unique ID to a replica so higher replica takes precedences
- Merge values together
- Record conflict and as user to resolve later

##### Custom conflict resolution logic

Run custom application logic

- On write: When detect conflict, call the conflict handler
- On read: All conflicting writes stored, when data is read, prompt user
  to resolve, then write back

There are some new automatic conflict resolutions

#### Multi-Leader Replication Topologies

**Replication topology**: communication paths where writers are
propagated

![](attachments/0766a843.png)

- All to all more fault tolerant
  - But replication messages "overtake" others
  - **version vectors** can be used

![](attachments/08fcabd6.png)

### Leaderless Replication

- Inspired by Amazon's Dynamo system
- Client send writes to several replicas while others, coordinator sends
  on behalf of client

#### Writing to DB when node is down

![](attachments/450ad7d7.png)

- Reading from node that went offline = potentially stale data
  - Read from multiple nodes.
  - Use version to solve conflicts

##### Read and repair and anti-entropy

- How to catchup after a replica is down?
  - Read repair: Writes new value to replica when stale data
  - Anti-entropy process: BG process that looks for differences
    - Helps increase durability (recovery when crash)

##### Quorums for reading and writing

- _n_ replica, _w_ nodes confirm write, _r_ nodes to consider successful
- **Qorum rules**: As long as _w_ + _r_ > n, we have up-to-date reading
- Common choice: n = odd, and `w = r= (n+1)/2`

Tolerate unavailability as follows:

- w < n, still process writes if node unavailable
- r < n, still process read if unavailable
- n = 3, w = 2, r = 2 - can have 1 node
- n = 5, w = 3, r = 3 - can have 2 node
- Don't need to distinguish how node fail, just as long as others return
- Error if fewer than w or r nodes

![](attachments/94e2ca15.png)

#### Limitations of Quorum Consistency

Quorums does not need to be majority, just as long sets of node has 1 overlap

Even with w + r > n edge cases where stale values returned:

- Sloppy quorum: w end up in different than r so no guarantee of overlap
- 2 writes occur concurrently. Can use last write wins
- Write happens with read, write reflected on some replicas
- Write suceeded on some replicas but failed on others, will not roll back
- Data restored from replica carrying old value break w
- Unlucky with **Linearizability and quorums**

w and r allow to adjust probability of stale values being read but not guarantee

##### Monitoring staleness

- No fixed order in which write are applied = harder to monitor

#### Sloppy Quorums and Hinted Handoff

- Some benefits of quorums discussed so far:
  - Tolerate failure of individual nodes w/o failover
  - Tolerate nodes slow bc don't have to wait n nodes to respond
  - Good for high availability & low latency, and can handle stale reads sometime
- But.....:
  - Not as fault-tolerant
  - Client cut off due to network interruption can cause client to no longer reach quorum

Trade off when client connect to some nodes

- Return error for request, or
- Accept write anyways and write to nodes that are not amongst the n nodes

  - AKA **sloppy quorum**. w and r still respected but different n nodes from usual
  - once fixed, node temporarily used return back to home: **hinted handoff**
  - Cannot be sure to read latest value
  - Assurance of durability but no guarantee read will see

##### Multi-datacenter operation

Send replicated writes to nodes in other datacenter but only quorum needed with
the local datacenter

#### Detecitng Concurrent Writes

- Dynamo-style db allow concurrent writes to same key
  ![](attachments/075d403c.png)
- Node2 thinks that its final value is B whereas other nodes is A
- To be **eventually consistent** replicas converge
  - DB handling is poor - app developer needs to know internals to prevent data loss

##### Last write wins (discarding concurrent writes)

- "recent" values are kept
  - **LLW last write wins** based on timestamp. Achieves eventual convergence
    but cost is durability. If losing data is unacceptable, LWW is poor choice for
    conflict
  - Safe way: ensure key is written once and immutable. Each write = unique key

##### "Happens-before" relationship and concurrency

- not concurrent: A operation depends on B or **casually dependent**
- concurrent: clients start operation on same key
  - Concurrent does not necessarily means happen at same time due to nature of
    clocks in distributed systems. Concurrency means when the two operations are
    unaware of each other

##### Capturing the happens-before relationship

![](attachments/9cec9983.png)
How it works: - Server maintain version number for every key and increment version # very time
key is written - When client reads a key, server returns all values not overwritten and latest number - Client writes a key, must include version from last read - When write w/ version number, can overwrite all values

##### Merging concurrently written values

- No daa is silently dropped but require clients to do more work
- Clients have to clean up after merging concurrent operations
  - Concurrent values = siblings
- Same issue with conflict resolution:
  - Can choose latest timestamp = lose data
  - Can take union
  - Can leave marker on version to indicate item has been removed so when merging
    siblings, application code can know not to include it in union - **tombstone**: deletion marker - Merge siblings in application code is complex & error prone. There are
    ways to make this automatic

##### Version vectors

- Single version number not enough w/ multiple replicas accepting writes concurrently
- Use version number per replica & keep track of version numbers from other replicas
- **version vector** version numbers from all replicas
- **dotted version vector** used in Riak
- These version vectors are sent to client and sent back to DB when value is written

### Summary

- Replication helps with:
  - **High Availability**: Keep system running even when one machine is down
  - **Disconnected operation**: Application continue to work even w/ network interruption
  - **Latency**: Geographically closer so faster
  - **Scalability**: Handle higher volume of reads
- Approaches to solve replication issues w/ concurrency etc:
  - **Single-leader replication**: All writes to single node
    - Easy to understand
    - No conflict resolution
  - **Multi-leader replication**: Client send write sto several leaders
    - Robust w/ faulty nodes, network interruptions, and latency spikes
    - Harder to reason and provide weak consistency
  - **Leaderless replication**: Send write to multiple nodes and reads to sever nodes to detect and correct
    nodes with stale data
- Strange effects caused by replication lag
  - **Read after write consistency**: User see data submitted themselves
  - **Monotonic reads**: After see data at a point, shouldn't see data from earlier
  - **Consistent prefix reads**: User should see data that makes causal sense: seeing question and reply in order
- Conflicts occur in multileader and leaderless replicaiton when multiple write happen
  - Algorithm that DB can use to determine if happen concurrently (version)
  
# 6. Partitioning

- Having multiple copies of same data on different nodes we will need to break up
the data into **partitions** aka **sharding**
- We want to partition because scalability: read distributed to many disks

## Partitioning and Replication
- Partition is combined with replication so copies stored on multiple nodes
- Each node acts as a leader for some partition and followers for others

![](attachments/29156868.png)

## Partitioning of Key-Value Data
- **Skewed** partition is when all load ends up in few partitions which are called **hot spot**
- Use KeyValue over random since random assignment reads have to look at all nodes

### Partitioning by Key Range
- Using ranges of keys
    - However, some ranges could be hot depending on access pattern
    
### Partitioning by Hash of Key
- Helps with skew and hots pots by generating uniformly distributed keys
- **Consistent hashing**: randomly chosen partition boundary to avoid need for central control or distributed consensus
- Loose range queries bc everything is scattered across partitions
- **compound primary key**: Use hashed first key to get partition and other column as range
    - If pk is `user_id, timestamp`, can query timestamp range efficiently
    
### Skewed Workloads and Relieving Hot Spots
- For hot specific keys, application and append random number to distribute. IE popular celebrity
    - Makes read difficult since need to look at all keys

## Partitioning and Secondary Indexes
- secondary indexes don't map neatly to partitions
- 2 types: **document-based partitioning** and **term-based partitioning**

### Partitioning Secondary Indexes by Document
- Each partition maintains its own local index
![](attachments/b00dad68.png) 

- Search will require **scatter/gather** which will search all paritions
    - tail latency amplification and expensive
    - Still used by MongoDB, Cassandra, ES
    - Recommend structure so secondary index queries from single partition but this is hard since could be multiple secondary index
    
### Partitioning Secondary Indexes by Term 
- Instead of each partition with secondary index, we can have global index
    - Need to partition the global index as well otherwise will be bottleneck
![](attachments/8cdbcdcd.png)
- **term-partitioned**: term determines partition of the index. ie `color:red`
- Good for range scans if partition by term
- Good for even distributed load if partition on hash of term
- Writes are slower and more complicated bc write now affect many partitions
    - term-partitioned index which is also distributed may not be reflected in index
    - GSI are async and eventual consistent
    - DDB fraction of a second delay

## Rebalancing Partitions
- Things change in a  DB
    - More query load so need more CPU
    - Dataset increase so need more disks and RAM
    - Machine fails so need another to take over
- **rebalancing**: process of moving load from one node in the cluster to another
- Requirements
    - helps with load evenly distributed
    - Accept read and writes while rebalancing
    - No more data than necessary to be moved
    
### Strategies for Rebalancing
#### Not to do: hash mod N
- Using mod is is not scalable since `mod 10` -> `mod 11` would reassign hash to different node

#### Fixed number of partitions
![](attachments/d13b163e.png)
- New node steals partition from the cluster
- Number of partition doesn't change but can (depends on DB)
- Choose high enough # of partition that strike balance between future growth and overhead of having too many partitions

#### Dynamic partitioning
