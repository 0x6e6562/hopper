Although this is not stated explicitly in the protocol, a channel is the smallest unit of parallelism in [AMQP][]. You _should_ bind an AMQP channel to a client thread, so that a channel is managed as a thread-local object. But what if you're using the same channel in multiple threads? The short answer: don't. This article summarizes some questions that have asked by people writing and using AMQP client libraries in a multithreaded fashion and provides some general advice about how to deal with parallelism in this context.

### Background

In the AMQP protocol, channels were designed to multiplex the transfer of unrelated messages from within a single physical transport, for example from within a single TCP connection. This mechanism allows the parallel execution of separate operations using a single AMQP client instance.

It could be argued that the ability to map multiple client threads or processes to a single connection introduces unnecessary complexity to the protocol. One could reason that the most effective way to apply potential parallelism in a client machine is to have a simple 1:1 relationship between a socket connection and a client thread. If you want to run multiple threads on a single client host, it is debatable whether the OS could manage the multiplexing of unrelated message streams more efficiently than an AMQP client library managing a single network connection.

A argument in favor of this proposal is that it would eliminate channels from the protocol and hence simplify the implementation of AMQP brokers. Whilst there is certain merit to this insight, in practice the advantage is likely to be minimal due to the following considerations:

* The connection state that is shared across multiple channels on the server side is small and relatively inconsequent;
* In any case, a broker needs to manage multiple unrelated streams of message passing;
* The broker needs to manage different conversational channels for each individual remote client.

So if the protocol were to be revised to remove the explicit concept of multiple channels within a single connection, it probably would not change much on the server side.

### Channel State

To avoid running into problems with multithreading using an AMQP client library, you need to understand that, from a protocol perspective, channels are effectively stateful . So as a client, you do need to ensure that AMQP commands are sent in a strict serial fashion from within a single channel. If channels were stateless with respect to the protocol, this would not be an issue. This requires consideration when sending commands that either:

* Require a synchronous response, such as exchange or queue declaration and binding, or;
* Consist of a header and a body such as the _BasicPublish_ command.

A client library needs to ensure that the various requests and responses of these commands are not interwoven in time - they must be sent sequentially.

For example, the _BasicPublish_ command contains a method header and a body for the message payload that are sent in separate wire frames. After receiving a header frame, the protocol flow expects the following frame to contain the message content. Using the [RabbitMQ][] as an example broker, if a frame for a different command were to be interwoven between the header and body frames, the broker would log the following error:

    COMMAND_INVALID - expected content header for class 60,
    got non content header frame instead in inspect
    
This symptom looks like an out of sequence issue that may indicate non-serial frame sending.

The current version of the specification offers a compromise to this restriction by allowing an application to use multiple channels which require no synchronization between each other.
 
### Implications For A Multithreaded Client

One of the obvious implications for this observation is to use a separate channel for each application thread. That way, you eliminate any race conditions that could occur between threads. However, when writing a client library, you still need to ensure that the client sends commands to the broker in a proper sequence.

In general, you can choose one of the following approaches to serializing command execution:

* The calling thread blocks for every request-response cycle and across the sending of header-body frames;
* The client library maintains a queue for outbound commands. When a command acknowledgement is received by the client, it is then free to drain any pending commands it has buffered up in the meantime;
* Entertain a mechanism whereby you register acknowledgment specific event handlers that are fired whenever a response comes off the wire;
* Buffer the sending of commands in a priority queue that is keyed on the AMQP class id of the method being buffered. This allows you to intersperse commands from different classes, but it becomes a bit complicated when you have contention between two instances of the same method (e.g. concurrent subscriptions to a queue).

### Alternative Approach: Look Ma, No Threads

An alternative approach to the threading issue is to not to allow threading in the client library at all. This approach was taken with the single threaded [py-amqplib][] AMQP client library for Python written by Barry Pederson.

### Lock Based Serialization

You could also consider avoiding contention by performing certain invocations in a fashion that is guaranteed to be globally atomic. For example, you could guard the transmission of a command with a mutex (e.g. a lock around the _BasicPublish_ method which sends three frames). Be aware that lock based solutions can be difficult to get right and there may be a considerable runtime overhead with this approach.

So say for example that you are pushing to one queue from multiple threads, and you have two threads trying to queue three frames (method, header, body) in order to publish a message. Under normal circumstances, those parts could get interleaved, so you would require a lock around the code inserting the three items into the queue.

Whilst it would be possible to just lock around basic publish, you would also need to make sure that every command within a channel is being sent serially. For correctness' sake, you need to ensure two things:

1. That a multi-part command like _BasicPublish_ is sent serially;
2. That synchronous RPCs are executed in order.

You could just use a lock for the first point, but how are you going to solve the second?

### Recommendations

Given all of the options listed above, I would opt for maintaining a queue per channel on the client side. That way, you enforce serialization without locking, it's serialization by blocking, basically. I would say that a queue is sufficient, since both a queue and lock exhibit serial semantics, which is what the problem statement requires.

There are practical examples of the locking and non-locking variants. To an extent, the RabbitMQ Java client takes the approach of locking the command execution within a single channel. On the other side, the RabbitMQ Erlang client takes the queueing approach. The reader should be advised to judge these design decisions in the context of the standard practices of the respective programming paradigms used to implement each client.

### Practical Implications For Threading

In general, you should strongly consider binding a client side thread to a server side channel. By doing so, a channel becomes a thread-local object, and hence, requires no synchronization.

However, there are circumstances where you may consider using the same channel in multiple threads.

Suppose that if you were to create a channel for each member of a pool of threads, you may have to re-declare queues and exchanges in each thread of the pool.

In this scenario, you could use the semantics of the AMQP protocol to your advantage. In AMQP, a declare command is nothing more than an assertion of the existence of an object. This makes a declaration an idempotent command, so it would be perfectly correct for each client side thread to make a declaration on the same queue or exchange name. This would avoid the necessity to maintain some kind of central declaration mechanism for queues, exchanges and bindings.

A further issue may be the wish to be able to re-use references to exchanges and queues among threads. If you wanted to do this, you would have to pre-declare each object in a sequential block before their subsequent usage in parallel code. However, in this case, you should remember than a reference to a queue or an exchange is nothing more than a name. In many programming languages, names are represented by strings, which, in turn, _can_ be treated as immutable and hence, are thread-safe objects.

### Efficiency Considerations

No technical advice should be followed literally without considering the concrete application scenario at hand. The goal of this article is to discuss the fundamental mechanics of parallelism within an AMQP broker in order to draw conclusions about how best to interact with it from a client perspective. Once your application is functioning correctly, performance may become an issue, and so therefore there are certain considerations that you could make. This section will discuss a few points of parallelism that may be relevant to how you use channels.

In the previous section, I stated that the declare commands were idempotent, so you could call them as many times as you like. On every invocation, they would guarantee the existence of the particular object in question. Whilst this is technically correct, your application _may_ experience better performance by avoiding redundant declarations. This is especially so in a clustered context, where the existence of a particular object has to be checked across a group of AMQP brokers. If your application can use domain specific knowledge to know that a particular object must already be in existence, then you could potentially avoid the overhead of explicitly declaring that object.

Whilst it may possible or even semantically desirable to start multiple queue consumers in the same channel, it should be pointed out that this could over-serialize the delivery of messages to the group of consumers. In general, you should be able to exploit more of the available parallelism in the broker if you opt for a channel per consumer strategy rather than starting multiple consumers in a single channel.

### Disclaimer

The observations about the protocol are based on version 0-8 of the AMQP specification. Subsequent revisions to the protocol may invalidate some of the conclusions drawn in this article.

[amqp]: http://www.amqp.org
[rabbitmq]: http://www.rabbitmq.com
[py-amqplib]: http://barryp.org/software/py-amqplib/