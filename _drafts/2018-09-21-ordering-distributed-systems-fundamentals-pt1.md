---
layout: post
title: "Ordering | Distributed Systems Fundamentals, Pt. 1"
description: "Lamport timestamps, vector clocks, and ordering in a distributed system"
category: Database
tags: ["database", "architecture", "distributed systems", "ordering"]
---

This is the first of what I hope is a long series of articles explaining the architecture
and implementation of [RansomDB](https://github.com/RansomDB/ransomdb), a distributed,
in-memory database that is suitible for both OLTP and OLAP workloads. I plan to write these
posts in conjunction with developing the related features in RandomDB.

From past experience, it is very difficult to take an application that was originally designed
to operate as a single synchronous unit and turn it into a distributed system. There are
certain distributed system patterns that are not terribly difficult to add to a monolithic
application, such as a work queue to process events whose results to not affect the main flow
of the application. However, in a distributed database, there are many aspects that need
coordination, and many transactions will probably need to be serviced by multiple nodes.

## Ordering in a Distributed System

One of the fundamental mechanisms in almost any distributed system is something that establishes
the ordering of events. Imagine this scenario: we have a timecard system that is keeps track of
the time that employees of an organization spend on various projects. There are 3 nodes in the
system (A, B, and C), and for the sake of this example, let's assume that any node should be able
to read or write any timecard. Now an employee, Rebecca, comes along and logs 4 hours on her project,
and all nodes receive the event and update their state accordingly. Later, Rebecca realizes that she
accidentally logged too much time and revises her timecard and removes one of her logged hours. Node
C receives this update, but its network cable had been unplugged, and the other nodes were not aware
of the change. Later, Rebecca's manager runs a report to get the time logged against the projects
she oversees. The request goes to Node A, and it needs to check to make sure that the value it has
is consistent with the values stored on the other nodes. Node B responds with 4 hours, and Node C
responds with 3, and we need to determine which is the correct (most recent) value.

### Lamport Timestamps

We can think of each node as having its own _logical clock_. Unlike a wall clock that tells the time
of day down to some specified level of granularity, a logical clock is a counter that is incremented
every time an operation occurs so that - within a specific node - we can always tell which operation
occurred before another.

Lamport timestamps, which are are named after their creator, [Leslie Lamport](https://en.wikipedia.org/wiki/Leslie_Lamport), generalize the concept of a logical clock to work in
a distributed system with multiple nodes that communicate asynchronously. Lamport timestamps allow
us to establish a logical clock between causally related events in a distributed system. That means
that if the outcome of event `e1` depends on the outcome of event `e0`, then the timestamp for `e0`
must be lower than the timestamp for `e1`. However, we do not need to impose a _total ordering_ for all
events because if the outcome of another event, `f0` does not depend in any way on `e0`, then we do
not care which one has the lower timestamp, because the end state of the system will be the same
regardless of whether `e0` or `f0` happened first.

The mechanics of a Lamport Timestamp are simple:

1. Every process maintains its own logical clock (counter) that it increments immediately every event
that it records
2. When a process sends a message to another process, it attaches the current value of its own clock
to the message
3. When a process receives a message, it compares its own timestamp to that of the message that it
received. If the timestamp on the incoming message is greater, then it sets its own timestamp to be
equal to that of the incoming message. Finally, the proccess increments is timestamp.

Given these very simple rules, we can ensure that if one event somehow effects another that we will
always be able to see them in the correct order. In terms of code, we could implement a simple
command-line app that allows us to see how this process works. We will use UDP to communicate