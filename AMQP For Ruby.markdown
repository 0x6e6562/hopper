[Aman Gupta][tmm1] has just completed a Ruby client implementation of [AMQP][] that is now available on [his Github repository][ruby-amqp]. This article describes the background of this project, the motivation behind a Ruby AMQP implementation and demonstrates some working code examples. This article will be of interest to you if you are looking for a simple Rubyesque API for interfacing with enterprise grade messaging.

### Motivation

Many Ruby authors have blogged on the benefits of asynchronous messaging in Ruby applications. Some interesting examples include [this article][nuby] from Geoffrey Grosenbach and a series of articles on [Jonathan Conway's blog][conway]. Asynchronous messaging gives Ruby the obvious potential to push back the barriers to scalability.

There are currently many approaches to asynchronous messaging in the Ruby community that have achieved significant levels of adoption. These include:

* ActiveMessaging;
* Starling;
* Sparrow;
* Memcached;
* STOMP

to name just a few. A common characteristic of these libraries is the way that they strive for simplicity, which is a very strong trait amongst Ruby developers in general.

AMQP, on the other hand, was primarily designed for low latency, high throughput and reliable messaging for financial applications. Ease-of-use was not necessarily the upmost design goal, although it would be subjective to say that AMQP is particularly complicated or simple. Let's just say for now that, out of the box, it is more involved than the average Rubyesque API.

Having said that, the AMQP model does have some nice benefits:

* It is designed for highly scalable deployment scenarios, so it will give you room to grow without having to re-architect your software;
* It defines a language-neutral wire format, so you can use it to integrate with other languages;
* It incorporates a robust and generalized message routing infrastructure.

Hence, the ultimate goal is to strike the golden middle path between the simplicity of a Ruby-orientated API and the architectural benefits of enterprise grade messaging.

### Background

Until now the only working Ruby implementation of AMQP has been a client that ships with Apache [QPid][]. [Dmitriy Samovskiy][dmitriy] has written [this article][dmitriy] that describes how to use this library.

Whilst this is a valid client side implementation of AMQP, it is quite cumbersome to use because you need to read and parse the XML specification of AMQP each time you use the client. Furthermore, it is not written in a Rubyesque fashion, which manifests itself in the way that exceptions are propagated to higher level code.

To be fair to QPid, as a messaging server developer, it is not an easy task to write and maintain clients in multiple clients. Apart from the increased effort of maintaining each extra client, you need to have some affinity with the current trends in each target language, otherwise you will fail to gain traction with the community of that language. Added to this is the fact that the more time you spend writing clients, the less time you spend improving the server, which is the thing that a server developer is more concerned about. Therefore, writing a client for a particular specification is best left to an active member of the community.

On the other hand, it is difficult for a community member to write an appropriate client side implementation, as they usually will not have the same overview of the specification as somebody who has written a server. Furthermore, they need to be convinced that it is worth spending time understanding the protocol and the effort involved in getting the client right.

So by implementing an AMQP client for Ruby, Aman has managed to successfully bridge two very different spheres of interest:

* Ruby developers who want to harness the benefits of asynchronous messaging in a simple fashion;
* AMQP broker developers who are primarily concerned with building scalable server side infrastructure.

The result is a Ruby Gem that provides a straightforward API that feels natural to Ruby developers whilst not compromising on the extensibility of a low level wire implementation of AMQP.

### Obtaining The Code

The AMQP library can be installed either as a Gem or by checking out the source code repository. To install it as a Gem, add Github as a source for Gems if you have not already:
    
    $ gem sources -a http://gems.github.com

Then you can install the Gem:

    $ sudo gem install tmm1-amp

If you prefer to get the source directly, you can just clone the git repository:
    
    $ git clone git://github.com/tmm1/amqp.git

### Booting The Server

This library was tested primarily with [RabbitMQ][], although it should be compatible with any server that is compliant with version 0-8 of the AMQP specification.
 
To use with RabbitMQ, you first need to start the server. You can either follow the friendly installation instructions on the RabbitMQ website, or, if you already have [Erlang][] installed on your system, you can just check out the server code from the Mercurial repository and run the server straight from a shell:
 
    $ hg clone http://hg.rabbitmq.com/rabbitmq-codegen
    $ hg clone http://hg.rabbitmq.com/rabbitmq-server
    $ cd rabbitmq-server
    $ make run
    
### Running The Examples

There are a number of scripts in the examples directory that you can look at and try out for yourself. I am going to briefly describe three:

* Ping Pong, where a message consumer responds to a ping message by sending a pong message;
* Clocks;
* Prime number generation. 


#### Ping Pong

This example demonstrates a peer-to-peer communication using an AMQP direct exchange. The following will run this example:

    $ ruby examples/pingpong.rb

The example uses Eventmachine to send ping messages to a well-known address (_one_): 

    EM.add_periodic_timer(1){
        amq.queue('one').publish('ping')
    }

To respond to these messages, a second peer is started:

    amq.queue('one').subscribe{ |headers, msg|
        log 'one', :received, msg, :sending, 'pong'
        amq.queue('two').publish('pong')
    }

This peer subscribes to the queue called _one_ and sends a pong message to another address every time it receives a ping.

This simple example demonstrates the simplicity of the high level Ruby API. Those with some knowledge of the AMQP protocol will notice that the low level broker connection management and channel multiplexing is hidden from the high level code. The library is designed so that each Ruby thread has its own channel to the broker, which is instantiated lazily.

Furthermore, it also shows how to leverage the default bindings for a direct exchange. By default, an AMQP broker must create a direct exchange for each queue with the same. In this fashion, messages can be routed directly to a queue without having to explicitly declare an exchange.
    
#### Clock

The second example shows how to use a fanout exchange in AMQP. A fanout exchange enables a very simple one-to-many communication pattern. 

In this example, a periodic timer event will be sent to a fanout exchange. This event can be consumed by multiple Ruby clients. The clock example can be run with the following command:

    $ ruby examples/clock.rb
    
A glance at the Ruby code will show that this example is a little more involved than the previous one, but it is still a very simple application:

    clock = MQ.new.fanout('clock')
    EM.add_periodic_timer(1){
        time = Time.now
        clock.publish(Marshal.dump(time))
    }

In contrast to the default direct exchange, a new AMQP exchange called _clock_ is created. After it has been successfully created, you can start sending timer events to it.

In order to receive events from a fanout exchange, a consumer needs to create a subscription queue and bind to the _clock_ exchange. This is a little more complicated than the pingpong example because, in this scenario, we are dealing with a 1:n communication pattern.

In this example, we are going to start two consumers that listen to different ticks from the event emitter. The first consumer creates a queue, binds it to the fanout exchange and prints the event every second: 

    amq = MQ.new
    amq.queue('every second').bind(amq.fanout('clock')).subscribe{ |time|
         log 'every second', :received, Marshal.load(time) 
    }

The second consumer creates its own queue and binds it to the same exchange. This consumer, however, is only interested in every fifth tick:

    amq = MQ.new
    amq.queue('every 5 seconds').bind(amq.fanout('clock')).subscribe{ |time|
        time = Marshal.load(time)
        log 'every 5 seconds', :received, \
        time if time.strftime('%S').to_i%5 == 0
    }

#### Prime Number Generation

The final example shows how to split up CPU-intensive jobs over multiple cores using an AMQP broker as a distribution mechanism. In this example, we compare a single-threaded program that checks every integer between 10000 and 11000 to see if it is a prime number with a program that distributes the same jobs via a queue mechanism.

The following examples were run on a Quad-Core Xeon X3220 @ 2.4 GHz running a 2.6.24 kernel:

    $ time ruby primes.rb 1 >/dev/null

    real 0m18.278s
    user 0m0.993s
    sys 0m0.027s

The single-threaded benchmark takes about 18 seconds to complete the task. As expected, most of the elapsed time is spent in a tight loop in user space. This is not making efficient usage of the the four cores that are available on the machine.

To divide the load up, worker processes can be used to consume requests for prime numbers from a queue:

    EM.fork(workers) do
      class PrimeChecker
        def is_prime? number
          log "prime checker #{MQ.id}", :prime?, number
          number.prime?
        end
      end
      MQ.rpc('prime checker', PrimeChecker.new)
    end

This demonstrates the server side of the RPC mechanism that is provided as a convenience mechanism by the Ruby AMQP library.

To create the prime number requests, the following code is used:

    EM.run{
        prime_checker = MQ.rpc('prime checker')
        (10_000..(10_000+MAX)).each do |num|
            prime_checker.is_prime?(num) { |prime|
                log :prime?, num, prime
                EM.stop_event_loop if ((@primes||=[]) << num).size == MAX
            }
        end
    }

This shows the other half of the same RPC abstraction, this time from the perspective of an RPC client.

If we introduce two worker processes that receive requests to compute the same prime number request, we observe that it actually increases the wall clock time required to complete the same job as the single-threaded program: 

    $ time ruby primes.rb 2 >/dev/null

    real 0m17.316s
    user 0m0.967s
    sys 0m0.053s

This indicates the overhead of distributing work using AMQP costs more than the benefit of exploiting more hardware parallelism.

However, if we start four workers instead of two we notice that the messaging overhead begins to pay off because it is making better use of the available hardware resources:

    $ time ruby primes.rb 4 >/dev/null

    real 0m8.229s
    user 0m1.010s
    sys 0m0.030s

If we increase the number of worker processes to eight or sixteen, the elapsed time is reduced even further:

    $ time ruby primes.rb 8 >/dev/null

    real 0m5.893s
    user 0m1.023s
    sys 0m0.050s

    $ time ruby primes.rb 16 >/dev/null

    real 0m5.601s
    user 0m0.990s
    sys 0m0.043s
    
In conclusion, this example demonstrates how easy it is to implement a simple work distribution mechanism for resource-intensive tasks.

### Outlook

Hopefully this article has demonstrated the simplicity and utility of using enterprise grade messaging with a Ruby program. This enables Ruby programmers to harness a paradigm that provides rich inter-language integration capabilities as well as providing the benefit of being able to easily access heavy duty strength messaging middleware. Hats off to Aman for bridging the gap between the Ruby and AMQP communities.

[conway]:http://www.jaikoo.com/
[nuby]:http://nubyonrails.com/articles/about-this-blog-beanstalk-messaging-queue
[erlang]:http://www.erlang.org/
[rabbitmq]:http://www.rabbitmq.com/
[dmitriy]:http://somic-org.homelinux.org/blog/2008/06/24/ruby-amqp-rabbitmq-example/
[qpid]:http://cwiki.apache.org/qpid/
[amqp]:http://amqp.org
[tmm1]:http://github.com/tmm1
[ruby-amqp]:http://github.com/tmm1/amqp/
