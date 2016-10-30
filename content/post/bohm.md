+++
title = "How to build a multi-version database that actually exploits multi-versioning"
date = "2016-10-25"
slug = "bohm-mvcc"
disqus_identifier="2016-10-25-bohm"
+++

This post will describe a serializable multi-version concurrency
control protocol, <a
href="http://jmfaleiro.com/pubs/multiversion-vldb2015.pdf">Bohm</a>,
that my advisor and I published at <a
href="http://www.vldb.org/2015/">VLDB 2015</a>. 

This post is divided into two parts, each can be read independently.
The first describes two important limitations of state-of-the-art
serializable multi-version concurrency control (MVCC) protocols. In
particular, it shows that several MVCC systems suffer from scalability
issues on multi-core hardware because they use centralized counters to
generate transaction timestamps. More importantly, we will see that
serializable MVCC protocols do not effectively exploit
multi-versioning to avoid read-write conflicts among update
transactions. The second half of the post will describe our
concurrency control protocol, Bohm, which addresses both these
limitations. Bohm is a concurrency control protocol designed for a
single-node main-memory resident database system. 

### Limitations of serializable MVCC

#### Timestamp generation
A multi-version database system maintains multiple copies, or
versions, of each record; each update to a particular record creates a
new version, while records' preceding values are preserved in older
versions.  As a consequence, a particular record may simultaneously
have multiple versions associated with it. 

<img src="/blog/img/mvcc_bohm/begin_ts.jpg" height="275" width="600">

Database systems typically distinguish between multiple versions using
timestamps. Both transactions and versions are assigned timestamps.
Each version has two timestamps associated with it, a begin- and
end-timestamp. Each transaction is assigned a read timestamp prior to its
execution. A version is visible to a transaction with read timestamp _ts_
if _ts_ lies between the version's begin- and end-timestamps. 
For instance, in the figure above, a transaction T<sub>0</sub>
attempts to credit a customer's savings account with $100. There exist
two versions of the savings record, one visible from time 0 to 10, and
another visible from 10 to infinity. The transaction is assigned a
timestamp of 12, which means that the second version is visible to
T<sub>0</sub>. 

<img src="/blog/img/mvcc_bohm/end_ts.jpg" height="275" width="600">

At the end of its execution, a transaction is assigned a write
timestamp, which determines the timestamps of records it
writes. Sticking with the previous example, T<sub>0</sub> is
assigned a write timestamp of 18. T<sub>0</sub>'s newly written out
version is assigned a 
begin timestamp of 18, and the previous version's end timestamp is
changed from infinity to 18.[^1]

Several MVCC protocols (both serializable _and_ non-serializable)
assign transactions _monotonically increasing_ timestamps. 
Monotonically increasing timestamps are
typically obtained using a global counter, which can be incremented by any 
thread or process in the system, which can turn 
into scalability bottlenecks on multi-core hardware.
Several DBMSs use a global 
counter as a source of timestamps, including PostgreSQL and
SQL Server's Hekaton main-memory DBMS.
It should be noted that both SQL Sever
Hekaton's and PostgreSQL's serializable protocols
rely on monotonicity for correctness --- the scalability bottleneck
cannot be removed by assigning transactions unique but non-monotonic
timestamps. A detailed discussion
on how timestamp monotonicity affects concurrency control
protocols is beyond the scope of this post. (I might write about it 
in a later post.) 

Our <a
href="http://jmfaleiro.com/pubs/multiversion-vldb2015.pdf">paper</a>
showed that monotonically increasing timestamps can significantly
impact the scalability of not just serializable MVCC protocols, but
even _non-serializable_ protocols such as Snapshot Isolation. In many
cases, our serializable protocol would significantly outperform
Snapshot Isolation despite providing _stronger_ correctness
guarantees.

#### Read-write concurrency between update transactions

Serializable MVCC systems execute read-only transactions against a snapshot of 
the database, and therefore isolate these reads from concurrent updates.
In the absence of read-only transactions, however, these systems do not 
efficiently exploit multi-versioning. As explained in a <a
href="/blog/post/mvcc-isolation">previous post</a>, an MVCC protocol
is prone to
serialization anomalies, such as write-skew, if concurrent reads and
writes among update transactions are completely decoupled.

Serializable MVCC systems deal with the problem by overly constraining
the execution of conflicting reads and writes between update
transactions. These systems either permit no additional
concurrency among conflicting reads and writes between update
transactions (as compared to single-version systems) or permit
limited read-write concurrency at the cost of maintaining extra meta-data on
reads. SQL Server's Hekaton main-memory engine and MySQL's InnoDB
belong to the first category. Hekaton uses a variant of optimistic
concurrency control in which transactions' reads are validated at the
end of their execution, while InnoDB transactions use record-level and
range locks on data. PostgreSQL's serializable snaphsot isolation
(SSI) protocol falls in the second category. SSI permits some
concurrency among conflicting reads and writes (reads and writes are
not completely decoupled), but requires reads to update shared
meta-data. Updating read meta-data isn't a big deal on disk-based
database systems, because transaction cost is invariably dominated by
disk seeks, and SSI's read meta-data is resident in main-memory. In a
main-memory database however, updating read meta-data can become
problematic, especially when multiple readers may compete to update
the same meta-data --- each read turns into an update.

### A new serializable MVCC protocol

Our new serializable MVCC protocol, Bohm, addresses both the above limitations
of conventional protocols. First, Bohm does not use a globally
updateable counter to assign transactions and records
timestamps. Second, Bohm ensures that reads _never_ block writes, and
that writes never cause a reading transaction to abort. Furthermore,
reads do not write any shared meta-data. 

These properties come at the expense of two limitations relative to
conventional serializable MVCC systems. First, Bohm requires
transactions to be submitted as one-shot stored procedures --- a
transaction's logic must be submitted to the database in its
entirety. Several main-memory database systems, including SQL Server
Hekaton and VoltDB, already make this assumption and use code-generation
techniques to convert SQL stored-procedures to C, C++, or assembly.
Second, Bohm requires that each transaction's write-set is
deducible prior to its execution. This does not imply that
transactions need to _pre-declare_ their write-sets; instead, a subset
of the transaction's logic may need to be speculatively executed in
order to determine its write-set.

Every serializable concurrency control protocol must ensure that it
schedules transactions in an order that is equivalent to _some_ serial
schedule. Conventional serializable MVCC protocols determine this
serialization as transactions execute, by acquiring locks or
optimistically validating transactions.  These protocols effectively
permit transaction to execute in parallel, and synchronize their
execution when required in order to produce serial equivalent
schedules. 

Bohm does the opposite. It _first_ imposes a total order on
transactions, relaxes this total order into a partial order
based on actual conflicts among transactions, and then finally
executes transactions based on the partial order. The total order
implicitly ensures that the schedule of transactions is serial
equivalent, while relaxing the total order into a partial order
ensures that the execution of non-conflicting transactions is not
unnecessarily constrained. Bohm executes 
transactions in two phases, a concurrency control and an execution
phase. The concurrency control phase totally orders transactions and
then relaxes the total order into a partial order. The execution phase
executes transactions' logic. 

#### Concurrency control phase

Bohm's concurrency control phase processes transactions in
batches. Transactions are totally ordered within a particular batch.
A transaction's position in the total order implicitly determines its
timestamp. This design circumvents the need for a global counter to
determine transactions' timestamps. Generating totally ordered batches
is embarassingly parallelizable; several threads can contribute to
producing different batches in parallel, and Bohm can select a batch
from each of these threads in round-robin.

At a high level, the concurrency control phase sequentially scans
through a particular transaction batch, and for every write that a
transaction must perform, writes out a version for the write. The rest
of this section describes how this is implemented in Bohm.

<img src="/blog/img/mvcc_bohm/cooperative.jpg" height="200" width="280">

Bohm divides the available threads in a system between its concurrency
control and execution phases. A set of long lived threads is dedicated
performing each phase. Bohm partitions the space of database objects
among a set of concurrency control threads. Concurrency control
threads sequentially scan through the a batch of transactions and
inspect each transaction's writeset. If a transaction performs a write
to an object in the concurrency control thread's partition, the thread
writes out a version corresponding to this write.  The figure above
shows an example in which the space of objects is divided among three
concurrency control threads, CC<sub>0</sub>, CC<sub>1</sub>, and
CC<sub>2</sub>. Transaction T<sub>200</sub> performs writes to objects
_a_, _b_, _h_, and _f_.  CC<sub>0</sub> will therefore write out a new
version for _a_, CC<sub>1</sub> for _b_ and _h_, and CC<sub>2</sub>
for _f_.

<img src="/blog/img/mvcc_bohm/cc_write.jpg" height="275" width="600">

The figure above shows how CC<sub>0</sub> creates a new version for
T<sub>200</sub>'s write to record _a_. _a_'s prior version's
end-timestamp is set to 200, while its new version's begin-timestamp
is set to 200. However, at this point, T<sub>200</sub> has not yet
executed, CC<sub>0</sub> therefore does not yet know the _value_ of
the new version will take.  Versions written out by concurrency
control threads therefore contain references to the transaction which must be
executed instead of a value. In the above example, the version created
by CC<sub>0</sub> will contain a reference to T<sub>200</sub>. 
In general, Bohm's concurrency control phase may
write out _several_ versions corresponding to the same record in a
single batch. None of the versions written out by concurrency control
threads will contain a value, only a reference to the transaction that
must be executed in order to obtain its value. 

We make two observations here. First, write-write
conflicts are always resolved by the same concurrency control
thread. If two different transactions write to the same record, then
the same concurrency control thread will create the versions
corresponding to these writes. Furthermore, since concurrency control
threads sequentially process transactions, the writes are always laid
out in an order that is consistent with transaction order. 

Second, unlike several other MVCC protocols, Bohm assigns transactions
a single timestamp, this timestamp determines the logical point in
time at which the transaction executes. The timestamp implicitly
determines the versions a transaction must read.  The versions written
by transactions are created by concurrency control threads. All
that remains is to fill in versions' values. This is performed by
the execution phase.

#### Execution phase

Bohm's execution phase takes in a batch of transactions as input after
it has been completely processed by the concurrency control phase.[^2]
Executing transactions is fairly straightforward. If a transaction T
needs to read a version whose value is not yet available, then T's
execution is paused and the transaction referenced in the version is
executed. This process continues recursively until the version's value
is obtained, at which point T can perform its read and resume its
execution. If the transaction referenced by T's version is already
executing, then T simply waits for the transaction to finish
executing. 

We make three observations here. First, reads by a particular
transaction never block conflicting writes. However, a read may have
to wait for a transaction to write out the value of a particular
version (via the recursive process described above). This is the
distinction between Bohm and snapshot isolation. Snapshot isolation
ensures that reads never block writes, and that writes never block
reads. However, these guarantees come at the expense of
serializability. Update transactions executing under snapshot
isolation are susceptible to the write-skew anomaly.

Second, write-write conflicts between transactions never lead
to aborts. This is because write-write conflicts are consistently
resolved _prior_ to transactions' execution. Several serializable MVCC
protocols abort transactions due to write-write conflicts among
concurrent transactions, including SQL Sever Hekaton and PostgreSQL
SSI. Even snapshot isolation aborts transactions due to write-write
conflicts. Aborting transactions due to conflicts wastes resources because
the database spends resources on a transaction which is eventually
aborted. These resources would have been better spent on executing
non-conflicting transactions.

Third, the process of laying out versions prior to transactions'
execution has the effect of relaxing the total order in a batch into a
partial order. The only case in which a transaction's execution is
constrained is if it has a read dependency on another
transaction. Read dependencies are precisely determined from a
transaction's timestamp and a version's transaction reference.
This is only possible because versions are laid out prior to
transactions' execution. 

### Wrapping up

Bohm addresses two important limitations of several widely deployed
serializable MVCC protocols; their use of contended global counters to
obtain transaction timestamps, and their inability to efficiently
exploit multi-versioning to obtain read-write concurrency among
conflicting update transactions.

If you end up reading the experimental section of the paper, you'll
notice that the multi-versioned protocols we implemented had a
significant disadvantage relative to single-version protocols on write heavy workloads.
Since
then, the excellent folks at the <a
href="https://db.in.tum.de/">Technical University of Munich</a> showed
how to eliminate the overhead of multi-versioning (relative to
single-version systems). These techniques can be directly used in Bohm. 



[^1]: While most MVCC protocols assign transactions a read and write timestamp, some protocols, such as timestamp ordering, assign transactions a single timestamp. 

[^2]: Bohm _pipelines_ the concurrency control and execution phases. When the concurrency control phase finishes processing a particular batch, it notifies the execution phase and immediately begins processing a new batch. 
