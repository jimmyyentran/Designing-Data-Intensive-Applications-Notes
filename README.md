- [4. Encoding and Evolution](#4-encoding-and-evolution)
  - [Dataflow through databases](#dataflow-through-databases)
  - [Dataflow Through Services: REST and RPC](#dataflow-through-services-rest-and-rpc)
    - [Web services](#web-services)
    - [Problem with RPCs](#problem-with-rpcs)
    - [Data encoding and evolution for RPC](#data-encoding-and-evolution-for-rpc)
  - [Message-Passing Dataflow](#message-passing-dataflow)
    - [Message broker](#message-broker)
  - [Summary](#summary)
- [5. Replication](#5-replication)
  - [Part II. Distributed Data](#part-ii-distributed-data)
  - [5. Replication intro](#5-replication-intro)
    - [Leaders and Followers](#leaders-and-followers)
    - [Problems with Replication Lag](#problems-with-replication-lag)
    - [Multi-Leader Replication](#multi-leader-replication)

# 4. Encoding and Evolution

## Dataflow through databases
- When older version of the application updates data previously written by a newer version of the application data may be lost
![5326b302.png](attachments/5326b302.png)

## Dataflow Through Services: REST and RPC
- Clients and servers
  - XMLHttpReqeust inside client-side JS so not HTML but something understandable by the client side
- **Service-oriented architecture (SOA)**: Decompose large application to smaller services by area of functionality
  - AKA **Microservices architecture**
- Services are similar to DB in sense that they allow submit and query data in specified format
- Key for service-oriented/microservice is to make application easeir to change by making services independently deployable and evolvable
  - Team owns service allows old and new versions running
  
### Web services
Whenever HTTP is used as underlying protocol
But webservices are used not obly on web but different context:
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
- Most common over HTTP though aims to be HTTP indipendent
- Complex standards
- **WSDL**: Web Services Description Language
  - Code-generation so client can access w/ local classes and methods
  - Not human-readable so heavy use on tools
- Fallen out of favor with smaller companies

### Problem with RPCs
**RPC**: Remote procedure calls
- Hide request to network serice to look same as function. AKA **Local transparency**
- Flawed since network calls are different than local function calls
  - local calls are predictable, network calls aren't
  - local calls return result, throw, or never return. Network calls also has return w/o result - timeout
  - Retry may have went through but responses are lost
    - **idempotence**: deduplication
  - Network calls are slower and latency is variable
  - Local calls pass in references to local memory. Network calls need to encode to bytes which are problematic for larger objects
  - Client and service can be implemented in different languages. Ex JS number max 2^53
  
#### Current directions for RPC
- It isn't going away.
- New RPC frameworks which don't hide it's a network call
  - Rest.li uses **promises** to encapsulate async calls and requrest multiple services in parallel
- **Service Discovery** Client find out which IP and port
- Binary encoding format = better performace than JSON over REST.
  - REST advantage is easier experimentation and debugging
  - More tools with REST
  
### Data encoding and evolution for RPC
- Servers update first, clients second. BC on request and FC on resp
- RPCService compat harder with cross org since no control over client (REST too)
- RPC no agreement on how API versioning work. Rest use version #

## Message-Passing Dataflow
**async message-passing systems**: Middle between RPC and DB.
- Similar to RPC in that req are delivered w/ low latency
- Similar to DB since message is not via direct network con but through message broker

Message broker adv (Kind of like SQS):
- Buffer if recipient is overlaoded
- Redeliver message if crash. Loss recovery
- Sender doesn't need to know IP and port of recipient
- One message to many recipient
- Decouples sender and client

Process is *one way* since sender do doesn't expect resp

### Message broker
- Old uses commerial enterprise
- New: RabbitMQ, ActiveMQ, HornetQ, NATS, and Apache Kafka
- Can chain consumer to ingest topic then publish another topic

#### Distributed actor frameworks

**actor model**: programming model for concurrency in a single process rather than dealing with threads.
- Communicate y sending async messages
- Mesasge delivery not guaranteeed
- *distributed actor frameworks*: Scale app accross multiple nodes. 
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
  - Programming language: specific encoding restricted to single  language and fail to provide FW and BW compat
  - Text: JSON, XML, CSV: Widespread, vauge of datatypes, numbers and binary string harder
  - Binary schema: efficeint encoding w/ FW and BW compat
    - Good for documentation
    - Code generation in statically typed
    - Now human readable
- Dataflow
  - DB
  - RPC and REST
  - Async message passing: Nodes communicate by sending messages encoded by sender, decode by recipient
 
# 5. Replication

## Part II. Distributed Data
**Scalability**: Read or write larger than single machine, need to spread across multiple
**Fault tolerance/high availability**: App continues to work if multiple machines go down
**Latency**: Servers close to user

Shared-memory architecture and shared-disk architecture both has downsides of cost when scaling.

**Share nothing architecture** Each machine runs independently as a node
- Distributed systems has constraints and tradeoffs
- Complexity of application and limits expressiveness

#### Replication vs Paritioning
**Replication**: Keeping same data on different nodes
- Provide redundancy: if nodes are unavailable, data can still be served from other
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

- **Leader based replication** aka active/assive or master-slave replication
  - 1 replica is the leader, write goes here
  - Other are followers. Leader send change to followers as part of replication log or change stream
  - Client need to read query from leader or followers. Write only on leader
  ![7f8fa9d8.png](:storage/bff7dba6-1552-4b2d-b716-59b56160bb36/7f8fa9d8.png)
- Relational and non relational uses this method. 
- Distributed message brokers like Kafka and RabbitMQ also uses this

#### Synch vs Async Replication
- pros of sync: guarantee of up-to-date copy o fdata
- con of sync: if follower doesn't erspond, leader will block all writes
- **semi-synchronous**: only one sync. if that sync follower is down or snow, will elect new sync follower

Leader based replication usually is async. But if fails before writes not replicated, can lose the write even if confirmed
- Advantage: leader can process writes even if followers behind
- useful if many followers and geo distribute

#### Setting up new Followers

Method without downtime:
1. Take a snapshot of leader point in time
2. Copy snapshot to new node
3. Follower connects to leader and request for changes since snapshot. Snapshot need to be associated with time
4. When follower is caught up, it process changes from leader

#### Handling node outages

**High availability**: Keep system running even single node fails

**Follower failure**: Catch-up recovery. Keep log of last transaction processed before fault. When connect again catch up on stream of data

**Leader failure**: One leader needs to be promoted to new leader. **Failover** clients reconfigured to consume from new leader.
- Determine leader failed: Node doesn't respond within time limit = dead
- Choose new leader: Election. Replica with most up-to-date 
- Reconfigure to use new leader: Client needs to send write to new leader. System needs to make sure old leader becomes a follower once it's up again

Failover issues:
- If async replication, new leader may not receive all writes from old leader before old leader failed. Most common: discard old leader's write
- Discarding write is dangerous if other storage system outside of DB needs to be coordinated. Ex. Redis and MySQL storage when MySQL db has new inconsistent leader
- **Split brain**: two nodes think they're leader and writes become conflicts
- Write timeout too short = unecessary failover. Too long = longer time to recover

#### Implementation of Replication Logs

##### Statement
**statement-based replication**: Write requests executed sent to followers

Possible ways to breakdown:
- nondeterministic function like NOW and RAND gets different valueon replicas
- Autoinc needs to be in order on each replica
- Triggers, stored procedures etc may work differently for each replica

##### Write ahead log shipping
- Append only log containin gall writes to DB
- Leader writes to log and send to followers
- Con: Logs are low level so replication is coupled with storage engine
  - If format changes, different version won't work between leader and followers
  - If replication protocol doesn't allow version mismatch = downtime
- Upgrade version by 1st upgrade followers, then failover to newly upgraded node as leader

##### Logical (row based) log replication
Different log format (**logical log**) for replication and for storage engine which allows decoupling from storage engine.

##### Trigger-based replication
Useful if only replicate subset of data, replicate one kind of DB from another, or resolution logic

**trigger** automatically run code when data change so can control where and how to replicate data

More error prone to bugs and limitations than DB built in replication

### Problems with Replication Lag
Leader-based replication good for small percent write and large read

*read scaling architecture*: increase capacity of read by adding more followers
- Works good with async
- Sync will be unreliable as single node down make system unavailable for writing

**eventual consistency**: Read on leader and follower is different since writes haven't been reflected in follower
- Delay could be fraction of second or minutes if max capacity or issue with network

#### Reading your own writes
![cdeffbc9.png](attachments/cdeffbc9.png)

**read-after-write consistency**: User will see updated submitted immediately
- Read user profile from the leader. Need way to know if user is looking at user's own profiel
- Track time of last update and recent changes pull from leader
- Client remember recent write, systems can make sure updates reflect until that timestamp
- Distrubted across geo is harder
- Cross devices
  - Client timestamp doesn't work bc one device doesnt know other device last update timestamp = metadata centralized
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
- **Consistent Prefix Reads**: write happens in order, anyone reading will see them appear in order
  - Occurs when DB are partitioned or sharded. IE User A in one shard, user B in another
  - Solution is to make writes that are causually related written in same partition but cannot be efficient

#### Solutions for Replication Lag
Solutions explored above can make application code complesx

Eventuall consistency is inevitable in scalable system

### Multi-Leader Replication
- More than one node accept writes
- **multi-leader configuration**: leader acts as follower and leaders

#### Use cases for MLR

##### Multi-datacenter operation
- leader in each datacenter and they both replicate each other's changes
- Performance: writes to local leader is faster so perceived performance is better
- Outage tolerance: data center operate independently
- network tolerance: async replication tolerate network problems better

**Conflict resolution**: Same data concurently modified in different datacenters and conflicts need to be resolved

Retrofitted features in many DB can lead to quirks

##### Clients with offline operation
Moblie devices with offline capability needs to be sync is in effect like 'leaders' with multi-leader replication process taken to the extreme