Parallelism & Fault Tolerance
=============================

Things You Should Know
----------------------

There is a collection of standard techniques that are useful in
parallel computation, for data and task assignment.

Data Assignment
---------------

In a distributed system, you often need a way to address data.
This is useful for creating partial locks over a data space, or
performing computation. Data addressing schemes are often referred
to as _partitioning_ or _sharding_.

There are two main methods for partitioning: hash and range.

Both methods rely on a _primary key_, sometimes simply called
the key. The key, and operations performed over it such as assignment
must be deterministic, so that any client application looking
for a record will always know where to look.

### Hash Partitioning Models

Hash partitioning is accomplished by hashing a record's key, and
using that hash to assign the record to a partition.

The mechanism used to perform hashing may be any hash method. It may
have strong uniqueness guarantees, such as a cryptographic hash
(MD5 or SHA) or it may have weaker uniqueness guarantees, such as
the `hashCode()` method in Java. Different database systems will rely
on different uniqueness guarantees based on their individual
access mechanisms.

Several distinct hashes of a record may be generated for fault tolerance
purposes, so that _m_ distinct replicas of a record may be assigned
and stored in multiple partitions. These distinct hashes may be generated
by using different hashing algorithms, or by appending or prepending
some known _m_ distinct values to the records in order to vary the final
hash. In such a system, metadata is typically stored with a 
record and consumed by clients in order to distinguish if it is a primary 
copy or a replica. This distinction can have important implications
on application consistency.

#### Modular Assignment

In modular assignment, a record is assigned to one of _n_ partitions.
This is done by finding _p_, which we define as the modulus-_n_ of the 
record's hash. The record is then assigned to the _p_-th partition.

Modular assignment is very common because it is easy to understand, easy to
implement, and it is highly scalable for a fixed number of partitions. These
qualities derive from the fact that only a single integer value is required to
uniquely address data. Modular hashing also allows an application to leverage
the parallel processing power of a cluster for many operations, such as ranged
access. 

If a system is to scale horizontally, however, data must be reassigned
whenever a partition is added. This can create significant complication
in many applications, such as databases, in order to ensure that data is 
available during rebalancing.

#### Ring Assignment

In ring assignment, each partition is identified by a random number that is
the size of the hash (so, for instance, if a hash is 20 bytes, as in SHA-1,
then each partition will be identified by a 20 byte number). Records are
assigned by the nearest floor or ceiling partition id's.

![Illustration of ring assignment][ring-assignment.svg]

For instance, if I had partition ids 0,2,6,7, and 10, and a record whose
hash value was 5, then my record would be assigned to the partition identified
by the number 2 (using the floor id).

Ring assignment is convenient because implementing horizontal scalability
within the system is much easier than in modular assignment, despite the fact
that implementing non-scalable versions of this assignment method are usually
much more complicated.

##### Hinted Handoff

Hinted handoff refers to techniques that enable rebalancing and availability
operations within a ring-assigned-hash-partitioned system; the term occurs
frequently in the documentation of databases such as Dynamo, Cassandra, and
Riak, but the actual meaning of the term varies slightly from database to
database.

In general, hinted handoff means that an adjacent partition may respond
to operations instead of it's neighbor. This may be necessary while a
ring is being rebalanced or in order to ensure application availability
during a failure mode.

Typically in hinted handoff, a client will begin searching for partitions
to respond in order of their adjacency to the unavailable partition.

Messaging
---------

In order for parallel processes to communicate with each other, they send
messages. In a typical scenario, a process is assigned a mutually exclusive
lock on some data (in memory, on disk, etc.) that represents its portion
of an application state. The process then receives and sends messages
in order to update its state.

There are many ways to perform messaging in a distributed system. Below,
I summarize some of the most common.

### Types of Messaging

#### Unicast

Unicast messages are messages between two processes. This messaging
pattern is probably the simplest of messaging patterns. Almost all
networking hardware implementations provide unicast messaging mechanisms.

#### Multicast

Multicast messages are messages that must be sent to multiple, or possibly
all processes within a distributed application. Multicast messages
are often tightly coupled to a specific hardware implementation of a network.

For instance, IP multicast messages use a special multicast IP address.

Many unicast messaging protocols are actually implemented thinly over a
multicast network fabric. For instance, a WiFi network pervades the entire
region of space that the radiation of devices' transmitting antennae is
able to reach. The radio in your personal laptop actually receives any
messages that are transmitted on the channel(s) that it is listening on.
It is up to your computer's networking stack and radio drivers to filter
out messages that are not specifically addressed to it.

#### Partition-Based

Partition-based messaging requires information about data or partition 
assignment in order to actually determine where to send a message.
Partition-based may be unicast or multicast.

Partitions may be constructed using hash or range partitioning methods.
Pieces of application state are assigned to partitions, and each partition
is assigned a process that is responsible for updating the state of the
partition.

Messages may then be directed to a specific partition. This is useful
when designing a database, for instance, because a single process may
maintain all the locks and state for a region of the database, which in turn
enables database scalability by avoiding global locks.

#### Gossiping

A gossip protocol involves 

### Resolving Conflicts

Conflict resolution is not a topic particular to distributed systems, but it
is one that gains special attention within the community due to the perhaps
increased likelihood of conflicts.

A conflict can arise in any application that accepts input from more than
one process or user at once. In a traditional web application, for instance,
two users may both log into a website and see that there is only one of a 
particular item left available for sale. If the two users attempt to purchase
the item, a conflict exists: neither user has been informed that the item
is no longer for sale, however only one of them may purchase it.

How to proceed next is complicated. There are both computational and business
reasons that you might not want to alert the user to a stock shortage. Perhaps
the database is very large, or eventually consistent; in such a situation, you
may not be able to report the discrepancy in a timely manner. Another issue
that may arise is that the store may be able to produce or order more items
with a minor delay, and would rather delay the user's order than to alert
them to the shortage and risk loosing the sale.

Several strategies for resolving and preventing conflicts are outlined below

#### Locking

One way to prevent the scenario above is to not allow two users to view the 
same item at the same time. This can be done through locking. In such a situation,
authoritative responsibility for maintaining a lock on a particular piece of
data is assigned to a process, usually using a partitioning scheme.

The Java Classpath's `java.util.concurrent` package contains implementations
of all the locks discussed below (and perhaps a few more) that are very
convenient to use within a single process. The Curator project also
contains implementations of most common lock types that may be leveraged by
multi-process applications.

Locks fundamentally involve three things:
    * Acquisition: acquiring a lock so that a resource can safely be read.
    * Waiting: a process or thread may use a lock to wait. This means pausing
      the process until 
    * 

There are many types of locks that provide different properties to an application.
To name just a few:
    - **Reentrant locks** allow the same process to re-acquire a lock. This is 
      helpful to prevent applications from hanging if they attempt to acquire
      the same lock multiple times. This is usually what is meant when someone
      refers simply to a "lock."
    - **Semaphores** Are locks that 
    - **Read-write locks** allow many processes to acquire the lock in read mode,
      and allow only one process to acquire the lock in write mode. This is
      commonly used to enforce multiple processes reading one piece of data at
      once, and to allow only one process to read or write the piece of data
      when being written. This semantic is actually enforced by the application
      and it's usage patterns of the locks, however, not by the data structure
      or the lock itself.
    - 

#### Vector Clocks

Vector clocks are a mechanism for keeping metadata about state change messages,
so that conflicts may be resolved by rule in application logic.

#### Paxos-ish

Paxos is a messaging protocol that guarantees total ordering of messages
within an application by allowing the 



Computation Models
------------------



### The (Original) MPI Model

One of the earliest implementations of a bridging model 

### Bulk Synchronous Parallel

