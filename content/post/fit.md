+++
title = "Why MongoDB, Cassandra, HBase, DynamoDB, and Riak will only let you perform transactions on a single data item"
description = "first description"
date = "2015-10-28"
slug = "fit-tradeoff"
disqus_identifier="2015-10-28-fit-tradeoff"
summary = "Testing FIT summary"
+++

_This is a joint post with Daniel Abadi, cross blogged here and at <a href="http://dbmsmusings.blogspot.com/2015/10/why-mongodb-cassandra-hbase-dynamodb_28.html">DBMS Musings</a>._

NoSQL systems such as MongoDB, Cassandra, HBase, DynamoDB, and Riak
have made many things easier for application developers. They
generally have extremely flexible data models, that reduce the burden
of advance prediction of how an application will change over
time. They support a wide variety of data types, allow nesting of
data, and dynamic addition of new attributes. Furthermore, on the
whole, they are relatively easy to install, with far fewer
configuration parameters and dependencies than many traditional
database systems.

On the other hand, their lack of support for traditional atomic
transactions is a major step backwards in terms of ease-of-use for
application developers. An atomic transaction enables a group of
writes (to different items in the database) to occur in an
all-or-nothing fashion --- either they will all succeed and be
reflected in the database state, or none of them will. Moreover, in
combination with appropriate concurrency control mechanisms, atomicity
guarantees that concurrent and subsequent transactions either observe
all of the completed writes of an atomic transaction or none of
them. Without atomic transactions, application developers have to
write corner-case code to account for cases in which a group of writes
(that are supposed to occur together) have only partially succeeded or
only partially observed by concurrent processes. This code is
error-prone, and requires complex understanding of the semantics of an
application.

At first it may seem odd that these NoSQL systems, that are so
well-known for their developer-friendly features, should lack such a
basic ease-of-use tool as an atomic transaction. One might have
thought that this missing feature is a simple matter of maturity ---
these systems are relatively new and perhaps they simply haven't yet
gotten around to implementing support for atomic transactions. Indeed,
Cassandra's "batch update" feature could be viewed as a mini-step in
this direction (despite the severe constraints on what types of
updates can be placed in a "batch update"). However, as we start to
approach a decade since these systems were introduced, it is clear
that there is a more fundamental reason for the lack of transactional
support in these systems.

Indeed, there is a deeper reason for their lack of transactional
support, and it stems from their focus on scalability. Most NoSQL
systems are designed to scale horizontally across many different
machines, where the data in a database is partitioned across these
machines. The writes in a (general) transaction may access data in
several different partitions (on several different machines). Such
transactions are called "distributed transactions". Guaranteeing
atomicity in distributed transactions requires that the machines that
participate in the transaction coordinate with each other. Each
machine must establish that the transaction can successfully commit on
_every other_ machine involved in the transaction. Furthermore, a
protocol is used to ensure that no machine involved in the transaction
will fail before the writes that it was involved in for that
transactions are present in stable storage. This avoids scenarios
where one set of nodes commit a transaction's writes, while another
set of nodes abort or fail before the transaction is complete (which
violates the all-or-nothing guarantee of atomicity).

This coordination process is expensive, both, in terms of resources,
and in terms of adding latency to database requests. However, the
bigger issue is that other operations are not allowed to read the
writes of a transaction until this coordination is complete, since the
all-or-nothing nature of transaction execution implies that these
writes may need to be rolled-back if the coordination process
determines that some of the writes cannot complete and the transaction
must be aborted. The delay of concurrent transactions can cause
further delay of other transactions that have overlapping read- and
write-sets with the delayed transactions, resulting in overall
"cloggage" of the system. The distributed coordination that is
required for distributed transactions thus has significant drawbacks
for overall database system performance --- both in terms of the
throughput of transactions per unit time that the system can process,
and in terms of the latency of transactions as they get caught up in
the cloggage (this cloggage latency often dominates the latency of the
transaction coordination protocol itself). Therefore, most NoSQL
systems have chosen to disallow general transactions altogether rather
than become susceptible to the performance pitfalls that distributed
transactions can entail.

MongoDB, Riak, HBase, and Cassandra all provide support for
transactions on a single key. This is because all information
associated with a single key is stored on a single machine (aside from
replicas stored elsewhere). Therefore, transactions on a single key
are guaranteed not to involve the types of complicated distributed
coordination described above.

Given that distributed transactions necessitate distributed
coordination, it would seem that there is a fundamental tradeoff
between scalable performance and support for distributed
transactions. Indeed, many practitioners assume that this is the
case. When they set out to build a scalable system, they immediately
assume that they will not be able to support distributed atomic
transactions without severe performance degradation.

This is in fact **completely false**. It is very much possible for a
scalable system to support performant distributed atomic transactions.

In a <a href="http://cs-www.cs.yale.edu/homes/dna/papers/fit.pdf">recent paper</a>, we published a new representation of the tradeoffs
involved in supporting atomic transactions in scalable systems.  In
particular, there exists a three-way tradeoff between fairness,
isolation, and throughput (FIT). A scalable database system which
supports atomic distributed transactions can achieve at most two out
of these three properties. Fairness corresponds to the intuitive
notion that the execution of any given transaction is not deliberately
delayed in order to benefit other transactions.  Isolation provides
each transaction with the illusion that it has the entire database
system to itself. In doing so, isolation guarantees that if any pair
of transactions conflict, then one transaction in the pair will always
observe the writes of the other. As a consequence, it alleviates
application developers from the burden of reasoning about complex
interleavings of conflicting transactions' reads and
writes. Throughput refers to the ability of the database to process
many concurrent transactions per unit time (without hiccups in
performance due to clogging).

The FIT tradeoff dictates that there exist three classes of systems that support atomic distributed transactions:

1. Those that guarantee fairness and isolation, but sacrifice throughput,
2. Those that guarantee fairness and throughput, but sacrifice isolation, and
3. Those that guarantee isolation and throughput, but sacrifice fairness


In other words, not only is it possible to build scalable systems with high throughput distributed transactions, but there actually exist two classes of systems that can do so: those that sacrifice isolation, and those that sacrifice fairness. We discuss each of these two alternatives below.

(Latency is not explicitly mentioned in the tradeoff, but systems that give up throughput also give up latency due to cloggage, and systems that give up fairness yield increased latency for those transactions treated unfairly.)

### Give up on isolation
As described above, the root source of the database system cloggage
isn't the distributed coordination itself. Rather, it is the fact that
other transactions that want to access the data that a particular
transaction wrote have to wait until _after_ the distributed
coordination is complete before reading or writing the shared
data. This waiting occurs due to strong isolation, which guarantees
that one transaction in a pair of conflicting must observe the writes
of the other. Since a transaction's writes are not guaranteed to
commit until after the distributed coordination process is complete,
concurrent conflicting transactions cannot make progress for the
duration of this coordination.

However, all of this assumes that it is unacceptable for transactions
writes to not be immediately observable by concurrent conflicting
transactions If this "isolation" requirement is dropped, there is no
need for other transactions to wait until the distributed coordination
is complete before executing and committing.

While giving up on strong isolation seemingly implies that distributed
databases cannot guarantee correctness (because transactions execute
against potentially stale database state), it turns out that there
exists a class of database constraints that can be guaranteed to hold
despite the use of weak isolation among transactions. For more details
on the kinds of guarantees that can hold on constraints despite weak
isolation, Peter Bailis's work on 
<a href="http://www.bailis.org/papers/ramp-sigmod2014.pdf">Read Atomic Multi-Partition</a>
(RAMP) transactions provides some great intuition.

### Give up on fairness
The underlying motivation for giving up isolation in systems is that
distributed coordination extends the duration for which transactions
with overlapping data accesses are unable to make
progress. Intuitively, distributed coordination and isolation
mechanisms overlap in time.  This suggests that another way to
circumvent the interaction between isolation techniques and
distributed coordination is to _re-order_ distributed coordination such
that its overlap with any isolation mechanism is minimized. This
intuition forms the basis of Isolation-Throughput systems (which give
up fairness).  In giving up fairness, database systems gain the
flexibility to pick the most opportune time to pay the cost of
distributed coordination.  For instance, it is possible to perform
coordination outside of transaction boundaries so that the additional
time required to do the coordination does not increase the time that
conflicting transactions cannot run. In general, when the system does
not need to guarantee fairness, it can deliberately prioritize or
delay specific transactions in order to benefit overall throughput.

<a href="http://www.cs.ucsb.edu/~sudipto/papers/socc10-das.pdf">G-Store</a> is a good example of an Isolation-Throughput system (which
gives up fairness).  G-Store extends a (non-transactional) distributed
key-value store with support for multi-key transactions.  G-Store
restricts the scope of transactions to an application defined set of
keys called a _KeyGroup_. An application defines KeyGroups dynamically
based on the set of keys it anticipates will be accessed together over
the course of some period of time. Note that the only restriction on
transactions is that the keys involved in the transaction be part of a
single KeyGroup. G-Store allows KeyGroups to be created and disbanded
when needed, and therefore effectively provides arbitrary transactions
over any set of keys.

When an application defines a KeyGroup, G-Store moves the constituent
keys from their nodes to a single leader node. The leader node copies
the corresponding key-value pairs, and all transactions on the
KeyGroup are executed on the leader. Since all the key-value pairs
involved in a transaction are stored on a single node (the leader
node), G-Store transactions do not need to execute a distributed
commit protocol during transaction execution.

G-Store pays the cost of distributed coordination prior to executing
transactions. In order to create a KeyGroup, G-Store executes an
expensive distributed protocol to allow a leader node to take
ownership of a KeyGroup, and then move the KeyGroup's constituent keys
to the leader node. The KeyGroup creation protocol involves expensive
distributed coordination, the cost of which is amortized across the
transactions which execute on the KeyGroup.

The key point is that while G-Store still must perform distributed
coordination, this coordination is done prior to transaction execution
--- before the need to be concerned with isolation from other
transactions. Once the distributed coordination is complete (all the
relevant data has been moved to a single master node), the transaction
completes quickly on a single node without forcing concurrent
transactions with overlapping data accesses to wait for distributed
coordination. Hence, G-Store achieves both high throughput and strong
isolation.


However, the requirement that transactions restrict their scope to a
single KeyGroup favors transactions that execute on keys which have
already been grouped. This is "unfair" to transactions that need to
execute on a set of as yet ungrouped keys. Before such transactions
can begin executing, G-Store must first disband existing KeyGroups to
which some keys may belong, and then create the appropriate KeyGroup
--- a process with much higher latency than if the desired KeyGroup
already existed.


### Conclusions
The fundamental reason for the poor performance of conventional
distributed transactions is the fact that the mechanisms for
guaranteeing atomicity (distributed coordination), and isolation
overlap in time. The key to enabling high throughput distributed
transactions is to separate these two concerns. This insight leads to
two ways of separating atomicity and isolation mechanisms. The first
option is to weaken isolation such that conflicting transactions can
execute and commit in parallel. The second option is to re-order
atomicity and isolation mechanisms so that they do not overlap in
time, and in doing so, give up fairness during transaction execution.