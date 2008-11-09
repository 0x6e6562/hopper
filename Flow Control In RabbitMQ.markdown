Recently I wrote [an article][scalability] about an item that the [RabbitMQ][] team has been working on lately, in which the scalability of routing was discussed. Mildly related to this is a further piece of work that has been carried out over the last few months - introducing producer flow control. This article describes what it is and how you can use it in RabbitMQ. It is assumed that the reader has some familiarity with the [AMQP][] model and also having some knowledge of [Erlang][] would be useful, but not essential. Readers who are not interested in the background of the protocol can jump forward to the practical side of this article.

### Background From A Protocol Perspective

The [0-8][] and [0-9][] versions of the AMQP protocol definition both state the following about flow control:

"_Flow control is an emergency procedure used to halt the flow of messages    from a peer. It works in the same way between client and server and is implemented by the Channel.Flow command. Flow control is the only mechanism    that can stop an over­producing publisher. A consumer can use the more elegant     mechanism of pre­fetch windowing, if it uses message acknowledgements (which usually means using transactions)._"

In contrast, AMQP version [0-10][], defines it thusly:

"_Flow control may be used to match the rate at which messages are sent to the available resources at the receiver. The receiver may be an AMQP server receiving messages published by a client, or a client receiving messages sent by an AMQP server from a queue. The same mechanism is used in both cases. In general, flow control uses the concept of credit to specify how many messages or how many octets of data may be sent at any given point without exhausting the receiver's resources. Credit is depleted as messages or data is sent, and increased as resources become free at the receiver. Pre-fetch buffers may be used at the receiver to reduce latency._"

So 0-10 has a broad notion that the semantics of flow control are symmetric for both ingress and egress. This may be an improvement on the previous version of the protocol, because part of the motivation behind 0-10 is to rectify conceptual inconsistencies of the 0-8/0-9 specifications.

However, since RabbitMQ is officially a 0-8 implementation, we will stick to those semantics for the purpose of this article.

Furthermore, RabbitMQ only implements the the server initiated variant of the _ChannelFlow_ command, i.e. the scenario where a broker wants to throttle a publisher.

### Why You May Need Flow Control

The _ideal_ steady state for a perfectly balanced message intermediary is one where the ingress rate equals the egress rate. The implication is that messages don't have to be queued up, they can just be delivered to the consuming application.

In reality, this rarely happens, at least not for indefinite periods of time. If the rate of production exceeds that of consumption, the difference will need to be buffered up in a FIFO fashion.

Any number of factors cause a discrepancy, for example:

* You are using the message broker to decouple production and consumption inter-temporally;
* The processing capacity on egress is somehow diminished, e.g. some consumers die;
* The cost of processing a message has changed over time or some messages have simply got bigger;
* More producers have been added to the ingress since the system was last in balance;
* Some producers have begun to send messages at a rate that is far greater than when the system was originally calibrated;
* Network congestion or broken links;

amongst many others. If this continues for a sustained period of time, the message broker will eventually run out resources if it has not taken any precautions to avoid this.

So there are a number of things that the intermediary could do to compensate for a disparity between publishers and consumers:

* Throttle the producers by sending a command instructing them to pause publication;
* Instead of buffering messages up in memory, overflow messages to disk;
* Alert an administrator to take steps to either decrease production or increase consumption;
* Simply cut off over-producing publishers;
* Silently discard messages;

This article discusses the first option as the (partial) implementation of the AMQP _ChannelFlow_ command.

### The Channel Flow Command

From a protocol perspective, when the broker decides that it is running out of resources to buffer an imbalance, it sends a channel.flow command out to every channel. This command contains a flag to indicate to a client whether content bearing AMQP methods can be sent or not. So if a client receives this command with the flag set to false, then the client should suspend sending any further content bearing methods (e.g. basic.publish), until such a time that the broker notifies the client that it is safe to resume. The notification of resumption is the opposite of the pause - the broker sends a channel.flow command with the flag set to true.

### How This Works in RabbitMQ

Under normal circumstances, RabbitMQ buffers each message that it has accepted for delivery in memory. If this is allowed to continue without draining any messages, RabbitMQ will not be able to allocate any more memory and it will crash.

To counteract this, RabbitMQ uses the [memsup][] facility which is part of the OS monitoring application of [OTP][]. Using memsup allows RabbitMQ to register a callback handler that is triggered when a high watermark for overall OS memory consumption has been detected.

When the memsup sets this high watermark alarm, RabbitMQ will send out the channel.flow command (with the active flag turned off) to every channel connected to it. The idea is that this will cause the producers to pause, giving the egress side of the application an opportunity to drain the overhang of messages. Once the memory has been garbage collected and the OS has reclaimed it, memsup will clear the high watermark alarm. This in turn prompts Rabbit to send out a new channel.flow command (with the active flag turned on) to each channel to indicate to them that message publication may resume.

### Testing Flow Control

It is quite easy to see this work in practice, though at the time of writing, this is only available in the latest source tree until RabbitMQ version 1.5.0 is released.

The *simplest* way to set the RabbitMQ broker up for this example is to get the source for the server and run it from a shell. To do this, you will need

* Erlang;
* Mercurial;
* Python

installed on your system, though if you want to skip the Mercurial clone, you can just get the latest source tarballs from http://hg.rabbitmq.com.

If you do use Mercurial, check out the RabbitMQ code generation module and the server module into the same directory and then compile the source code:

    $ hg clone http://hg.rabbitmq.com/rabbitmq-codegen
    $ hg clone http://hg.rabbitmq.com/rabbitmq-server
    $ cd rabbitmq-server
    $ make -j
    python codegen.py header ../rabbitmq-codegen//amqp-0.8.json > include/rabbit_framing.hrl
    python codegen.py header ../rabbitmq-codegen//amqp-0.8.json > include/rabbit_framing.hrl
    python codegen.py body   ../rabbitmq-codegen//amqp-0.8.json > src/rabbit_framing.erl
    python codegen.py body   ../rabbitmq-codegen//amqp-0.8.json > src/rabbit_framing.erl
    erlc -I include -o ebin -Wall -v +debug_info -Duse_specs src/buffering_proxy.erl
    erlc -I include -o ebin -Wall -v +debug_info -Duse_specs src/rabbit.erl
    .....

The same process applies if you opt to use the source tarballs instead - just unpack them into the same parent directory.

After successful compilation, all you have to do is run the server:

    $ make run
    NODE_IP_ADDRESS= NODE_PORT= NODE_ONLY=true LOG_BASE=/tmp  RABBIT_ARGS=" -s rabbit" MNESIA_DIR=/tmp/rabbitmq-rabbit-mnesia ./scripts/rabbitmq-server
    Erlang (BEAM) emulator version 5.6.4 [source] [smp:2] [async-threads:30] [kernel-poll:true]

    Eshell V5.6.4  (abort with ^G)
    (rabbit@xlr8)1> RabbitMQ %%VERSION%% (AMQP 8-0)
    ...
    Logging to "/tmp/rabbit.log"
    SASL logging to "/tmp/rabbit-sasl.log"

    starting database             ...done
    starting core processes       ...done
    starting recovery             ...done
    starting persister            ...done
    starting builtin applications ...done
    starting TCP listeners        ...done

    broker running
    
Leave this shell where it is for now, we will come back to it later.
    
The RabbitMQ Java client contains a test class called _MulticastMain_ which, amongst other things, can be used to flood the broker with messages. To see this, run it with the following command line arguments:

    $ java -Xmx512m -Xms256m -cp $CP com.rabbitmq.examples.MulticastMain -s 10000 -n 1
    starting consumer #0
    starting producer #0
    recving rate: 300 msg/s, min/avg/max latency: 6519/458396/835529 microseconds
    sending rate: 5410 msg/s
    recving rate: 352 msg/s, min/avg/max latency: 836990/1362443/1741541 microseconds
    sending rate: 5781 msg/s
    recving rate: 330 msg/s, min/avg/max latency: 1750540/2222147/2688223 microseconds
    sending rate: 5901 msg/s
    recving rate: 2020 msg/s, min/avg/max latency: 2687705/3242368/3338921 microseconds
    ....

To see various options that you can pass to MulticastMain, pass in the --help argument to print out the help. In this scenario, we are using these options:

     -n,--ctxsize <arg>     consumer tx size
     -s,--size <arg>        message size    

By default MulticastMain will start a single producer thread and a single consumer thread, but this can be changed using the CLI options. In this example, the consumer transaction size of one _should_ allow the producer to send messages at rate quicker than the consumer can consume them.

If there is ample memory available to the broker, testing flow control may be a tedious task, because by default, the high watermark is set to 95% of system memory, which might take a long time to exhaust (or your client runs out of memory first).

However, we can use a cool feature of Erlang/OTP to cut to the chase. To do this, go back to the shell where you started the broker and press Enter. This should give you a prompt in the Erlang shell:

    .....
    broker running

    (rabbit@yourhost)1>

At the prompt, you can use the memsup module to set high watermark to a value that is sufficiently low enough to trigger the alarm on your system, e.g. 5%:

    (rabbit@yourhost)1> memsup:set_sysmem_high_watermark(0.05).
    ok

Depending on how memory is currently available on your system, you should be able to see the high watermark alarm being set in the log file:

    $ tail -f /tmp/rabbit.log
    
    =INFO REPORT==== 9-Nov-2008::15:13:31 ===
        alarm_handler: {set,{system_memory_high_watermark,[]}}

Turning your attention back to the client log output, you should be able to see that the client has stopped sending messages by the fact that for a small period, only statements from the consumer thread are logged (until the backlog has been drained):

    sending rate: 6153 msg/s
    recving rate: 245 msg/s, min/avg/max latency: 2831384/3327989/3787075 microseconds
    sending rate: 5523 msg/s
    recving rate: 189 msg/s, min/avg/max latency: 3801915/4509938/4763547 microseconds
    recving rate: 4133 msg/s, min/avg/max latency: 4757701/4858617/4981038 microseconds
    recving rate: 4327 msg/s, min/avg/max latency: 4972126/5124176/5251434 microseconds
    recving rate: 4309 msg/s, min/avg/max latency: 5251628/5416321/5588283 microseconds
    recving rate: 4356 msg/s, min/avg/max latency: 5585425/5688730/5835853 microseconds
    recving rate: 4533 msg/s, min/avg/max latency: 5833382/5983939/6136073 microseconds
    recving rate: 4588 msg/s, min/avg/max latency: 6072685/6139336/6232172 microseconds

After the backlog has been drained, the memory alarm will be cleared as indicated in the server log file:

    =INFO REPORT==== 9-Nov-2008::15:14:29 ===
        alarm_handler: {clear,system_memory_high_watermark}

After that, RabbitMQ will inform the producer that it can resume publication.

If you try this out yourself, be aware that the actual values that cause RabbitMQ to issue a channel.flow command will depend on your system memory resources, so your mileage may vary. Having said this, you could then proceed to find out where the memory "sweet spot" is by a process of trying to find out what percentage of OS memory will trigger the alarm. This may be of little practical relevance, but at least it gives you a feel for the mechanics at work on your particular setup.

### Limitations

RabbitMQ does not (yet) implement the _BasicQos_ command which would allow more grained control of the egress flow (as opposed to relying on back pressure to the socket). This can be seen by the fact that the consumer thread uses the _QueueingConsumer_ component of the RabbitMQ Java client. The _QueueingConsumer_ buffers up all of the incoming messages in the address space of the consumer process. This has a few implications:

* It absorbs the back pressure that the consumer would otherwise exert on the socket to help with flow control;
* This uses a lot of memory which can either exhaust the client heap or cause the garbage collector to decide that it cannot reclaim memory quick enough and end the process.

### Other Options And Extensions

The nature of the _ChannelFlow_ command is that of a safety valve - if the broker thinks it can't handle the current ingress rate, it decides to pull the emergency brake. However, there are other options that could be implemented _outside_ of the broker to achieve the same goal:

* Set the immediate flag for delivery when publishing messages - this will avoid any queuing whatsoever;
* Implement a client that listens for the _ChanelFlow_ command and performs some form of application or infrastructure resource allocation to boost the egress rate;
* Set the TTL header field when sending a message to a value that will cause the broker to expire the message if it does not get delivered before too long.

The first option is available via the _BasicPublish_ command already.

The second option could be a callback handler in the client library that provides an event driven mechanism to deploy more resources. At the time of writing this article, a callback interface was not implemented in the RabbitMQ Java client, but it has been scheduled in the product roadmap.

The third option has not yet been implemented in the RabbitMQ broker, but is on the roadmap.

Furthermore, one can think of extensions to the broker that can help alleviate the general issue:

* Overflow messages to disk;
* Define a maximum queue depth as a global variable or on a per-queue basis;
* Implement last image caching, whereby messages that have been semantically superseded by newer messages are automatically discarded.

[erlang]:http://www.erlang.org
[rabbitmq]:http://www.rabbitmq.com
[scalability]:http://hopper.squarespace.com/blog/2008/11/7/measuring-scalability.html
[amqp]:http://www.amqp.org
[0-10]:http://jira.amqp.org/confluence/download/attachments/720900/amqp.0-10.pdf?version=1
[0-9]:http://jira.amqp.org/confluence/download/attachments/720900/amqp0-9.pdf?version=1
[0-8]:http://jira.amqp.org/confluence/download/attachments/720900/amqp0-8.pdf?version=1
[memsup]:http://www.erlang.org/doc/man/memsup.html
[OTP]:http://www.erlang.org/doc/design_principles/part_frame.html