<!--
Copyright (c) 2007-2019 Pivotal Software, Inc.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Reliability Guide

## <a id="overview" class="anchor" href="#overview">Overview</a>

This guides provides an overview features of
RabbitMQ, AMQP 0-9-1 and other supported protocols related to data safety.
They help application developers and operators achieve <i>reliable delivery</i>,
that is, to ensure that messages are always delivered, even encountering failures
of various kinds.

Data safety is a joint responsibility of RabbitMQ nodes, [publishers](/publishers.html)
and [consumers](/consumers.html). Therefore, this guide provides an overview of
topics imported to each part of a messaging-based system.

## <a id="what-can-fail" class="anchor" href="#what-can-fail">What Can Fail?</a>

Messaging-based systems are distributed by definition and can fail in
different, and sometimes subtle, ways.

Network connection problems and congestion are probably the most common class of failure.
Not only can networks fail, [firewalls can interrupt connections](/heartbeats.html#tcp-proxies)
they consider to be idle, and network failures [take time to detect](/heartbeats.html).

In addition to connectivity failures, the server and client
applications can experience hardware failure (or software can crash)
at any time. Additionally, even if client applications keep running,
logic errors can cause [channel](/channels.html#error-handling) or [connection errors](/connections.html#error-handling) which force the
client to establish a new channel or connection and recover from the
problem.

## <a id="connection-failures" class="anchor" href="#connection-failures">Connection Failures</a>

In the event of a network connection failure between a client and RabbitMQ node,
the client will need to establish a new connection to the broker. Any channels opened on the
previous connection will have been automatically closed and these will
need re-opening too.

In general when connections fail, the client will be informed by the
connection throwing an exception (or similar language construct).

Most client libraries provide a feature that automatically recovers from connection
failures. For cases where this opinionated recovery is not suitable, application
developers can implement their own recovery by defining connection failure
event handlers. See client documentation, such as the [Java](/api-guide.html)
and [.NET client guides](/dotnet-api-guide.html), to learn more.

### <a id="confirms" class="anchor" href="#confirms">Acknowledgements and Confirms</a>

When a connection fails, messages may be in transit between client and
server - they may be in the middle of being decoded or encoded on either side,
sit in TCP stack buffers, or be in flight on the wire. In such events
messages in transit will not be delivered — they will
need to be retransmitted. [Acknowledgements](/confirms.html) let the server and
clients know when to do this.

Acknowledgements can be used in both directions - to allow a consumer
to indicate to the server that it has received and/or processed a delivery
and to allow the server to indicate the same thing to the
publisher. Both are known as consumer acknowledgements and publisher confirms.

While TCP ensures that packets have been delivered to connection peer, and will
retransmit until they are, that only handles failures at the network
layer. Acknowledgements and confirms indicate that messages have been
received **and acted upon** by the peer application.
An acknowledgement signals both the receipt of a message, and a transfer of ownership where
the receiver assumes full responsibility for it.

Acknowledgements therefore have semantics. A consuming application
should not acknowledge messages until it has done whatever it needs to
do with them: recorded them in a data store, forwarded them on, or perform
any other operation. Once it does so, the broker is free
to mark the delivery for deletion.

Similarly, the broker will confirm messages once it has taken
responsibility for them. The details are covered in the [Acknowledgements and Confirms guide](/confirms.html).

Use of acknowledgements guarantees **at least once**
delivery. Without acknowledgements, message loss is possible
during publish and consume operations and
only **at most once** delivery is guaranteed.

## <a id="heartbeats" class="anchor" href="#heartbeats">Detecting Dead TCP Connections with Heartbeats</a>

In some types of network failure, packet loss can mean that
disrupted TCP connections take a moderately long time (about 11
minutes with default configuration on Linux, for example) to be
detected by the operating system. AMQP 0-9-1 offers a
<i>heartbeat</i> feature to ensure that the application layer
promptly finds out about disrupted connections (and also
completely unresponsive peers). Heartbeats also defend against
certain network equipment which may terminate "idle" TCP
connections. See <a href="heartbeats.html">Heartbeats</a> for details.

## <a id="broker-side" class="anchor" href="#broker-side">Data Safety on the Broker Side</a>

In order to avoid losing messages in the broker, queues and messages must be able to cope with
broker restarts, broker hardware failure and <i>in extremis</i> even
broker crashes.

To ensure that messages and broker definitions survive restarts, we
need to ensure that they are on disk. The AMQP standard has a concept
of durability for exchanges, queues and of persistent messages,
requiring that a durable object or persistent message will survive a
restart. More details about specific flags pertaining to durability
and persistence can be found in the
<a href="/queues.html">Queues guide</a>.

## <a id="clustering" class="anchor" href="#clustering">Clustering and Message Replication</a>

If we need to ensure that our broker survives hardware failure, we can
use RabbitMQ's clustering. In a RabbitMQ cluster, all definitions (of
exchanges, bindings, users, etc) are mirrored across the entire
cluster. Queues behave differently, by default residing only on a
single node, but optionally being mirrored across several or all
nodes. Queues remain visible and reachable from all nodes regardless
of where they are located.

Mirrored queues replicate their contents across all configured cluster
nodes, tolerating node failures seamlessly and without message loss
(although see <a href="ha.html#unsynchronised-mirrors">this note on
unsynchronised mirrors</a>). However, consuming applications need to be
aware that when queues fail their consumers will be cancelled and they
will need to reconsume - see <a href="ha.html#behaviour">the
documentation</a> for more details.


## <a id="publisher-side" class="anchor" href="#publisher-side">Data Safety on the Publisher Side</a>

When using confirms, producers recovering from a channel or connection
failure should retransmit any messages for which an acknowledgement
has not been received from the broker. There is a possibility of
message duplication here, because the broker might have sent a
confirmation that never reached the producer (due to network failures,
etc). Therefore consumer applications will need to perform
deduplication or handle incoming messages in an idempotent manner.

### <a id="routing" class="anchor" href="#routing">Ensuring that Messages are Routed</a>

In some circumstances it can be important for producers to ensure that
their messages are being routed to queues (although not always - in
the case of a pub-sub system producers will just publish and if no
consumers are interested it is correct for messages to be dropped).

To ensure messages are routed to a single known queue, the producer
can just declare a destination queue and publish directly to it. If
messages may be routed in more complex ways but the producer still
needs to know if they reached at least one queue, it can set the
<code>mandatory</code> flag on a <code>basic.publish</code>, ensuring
that a <code>basic.return</code> (containing a reply code and some
textual explanation) will be sent back to the client if no queues were
appropriately bound. See the [Publishers guide](/publishers.html) for details.

Producers should also be aware that when publishing to a clustered node,
if one or more destination queues that are bound to the exchange have
mirrors in the cluster, it's possible to incur delays in the face of
network failures between nodes, due to flow control between replicas
and the queue master replica. See <a href="nettick.html">inter-node heartbeat guide</a> for
more details.


## <a id="consumer-side" class="anchor" href="#publisher-side">Data Safety on the Consumer Side</a>

In the event of network failure (or a node failure), messages can be
duplicated, and consumers must be prepared to handle them. If
possible, the simplest way to handle this is to ensure that your
consumers handle messages in an idempotent way rather than explicitly
deal with deduplication.

If a message is delivered to a consumer and then requeued (because it
was not acknowledged before the consumer connection dropped, for
example) then RabbitMQ will set the <code>redelivered</code> flag on
it when it is delivered again (whether to the same consumer or a
different one). This is a hint that a consumer <i>may</i> have seen
this message before (although that's not guaranteed, the message may
have made it out of the broker but not into a consumer before the
connection dropped). Conversely if the <code>redelivered</code> flag
is not set then it is guaranteed that the message has not been seen
before. Therefore if a consumer finds it more expensive to deduplicate
messages or process them in an idempotent manner, it can do this only
for messages with the <code>redelivered</code> flag set.

### <a id="cancel-notification" class="anchor" href="#cancel-notification">Consumer Cancel Notification</a>

Under some circumstances the server needs to be able to cancel a
consumer - since the queue it was consuming from has been deleted, or
has <a href="ha.html#behaviour">a new queue master was promoted</a>. In this case the
consumer should consume again but be aware that it may see messages
again which it has already seen.

Note that consumer cancel notification is a RabbitMQ extension to
AMQP 0-9-1, and as such may not be supported by all clients.

### <a id="unprocessable-deliveries" class="anchor" href="#unprocessable-deliveries">Unprocessable Deliveries</a>

If a consumer determines that it cannot handle a message then it
can <i>reject</i> it using <code>basic.reject</code>
(or <code>basic.nack</code>), either asking the server to requeue it,
or not (in which case the server might be configured
to <a href="dlx.html">dead-letter</a> it instead.

## <a id="consumer-side" class="anchor" href="#publisher-side">Federation and Shovel</a>

Rabbit provides two plugins to assist with distributing nodes over
unreliable networks: <a href="federation.html">Federation</a> and
the <a href="shovel.html">Shovel</a>. Both are implemented as AMQP
clients, so if you configure them to use confirms and
acknowledgements, they will retransmit when necessary. Both will use
confirms and acknowledgements by default.

When connecting clusters with Federation or the Shovel, it is
desirable to ensure that the federation links and shovels tolerate
node failures. Federation will automatically distribute links across
the downstream cluster and fail them over on failure of a downstream
node. In order to connect to a new upstream when an upstream
node fails, you can specify multiple redundant URIs for an upstream,
or connect via a TCP load balancer.

When using the shovel, it is possible to specify redundant brokers in
a source or destination clause; however it is not currently possible
to make the shovel itself redundant. We hope to improve this situation
in the future; in the mean time a new node can be brought up manually
to run a shovel if the node it was originally running on fails.