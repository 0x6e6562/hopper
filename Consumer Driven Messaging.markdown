A fair while ago, Kirk Wylie wrote [an article][kirk] about some of the conceptual mistakes he felt were made in the specification of the [AMQP][] protocol. Amongst other observations, he made the point that a producer _should_ require no knowledge of the routing semantics of an exchange in order to send a message. He observes that this currently may not be the case, and hence questions the consistency of the model.

### Decoupling Producers And Consumers

One of the fundamental concepts of the AMQP model is that a consumer decides what messages it wants to receive, not a producer. Although this is not explicitly stated in the AMQP specification, one of the main pieces of functionality that a compliant AMQP broker is required to deliver, is consumer driven message routing. This paradigm has the intention to completely decouple message producers from any downstream endpoints or recipients. The advantage of this approach is the flexibility it gives to the architect a message driven system.

### Problem Statement

However, Kirk postulates that there may have a shortsighted design decision which could lead to breaking this fundamental decoupling of production and consumption.

For readers not intimately familiar with the AMQP specification, there are four main types of message exchanges defined in the 0-9 version of the protocol:

* Direct exchange, which routes messages based on matching a particular key;
* Fanout exchange, which routes messages unconditionally to any bound queue;
* Topic exchange, which implements hierarchal pattern matching;
* Headers exchange, which implements a match based on the contents of a map.

Hence the type of exchange is of central importance to the routing logic that will be applied to each and every message sent to an exchange.

Kirk's argument follows the logic that since the declaration of an exchange requires an exchange type, the producer is implicitly forced to impart knowledge of the subsequent routing on to the exchange. This is because the producer would have to declare the existence of an exchange in order to publish to it. Hence, by having to declare what type of exchange it has to be, the producer is effectively coupled to the routing logic, which is exactly what the AMQP model was trying to avoid.

### The Nature Of Exchange Declaration

The assumption upon which the previous conjecture was based was the fact that you have to declare an exchange before you can start sending messages to it. To be precise, to successfully send a message to an exchange, it must exist. The sender does not necessarily have to have created the exchange instance itself.

Moreover, object declaration in AMQP is akin to an assertion of an object's existence - by declaring an exchange (or a queue for that matter), you are basically saying to the broker, "I assert the existence of object X, if it does not exist, make it so".

### Must A Producer Declare An Exchange?

The short answer is no, it doesn't have to. It can, if it wants to. Since exchange declaration is idempotent, from the broker's perspective, it doesn't matter either way.

Having said this, if a producer sends a message to an non-existent exchange, this will cause an error. If the producer were to have declared the exchange before it published a message, this would not raise an error. So one would conclude that it is better for the producer to declare an exchange, lest they be unable to send a message to it because it doesn't exist.

Following on from the original argument, this must lead to a semantic coupling, because in order to declare the exchange, the producer must supply the exchange type.

Let's just say for arguments sake, that a producer does decide to pre-declare the exchange, and that exchange did _not_ previously exist, so the broker will create a new exchange object on the producer's request. Then the producer is free to fire and forget messages at that exchange - this will not raise an error, because the exchange is guaranteed to exist.

However, if there are no queues bound to that exchange, any messages sent to it will be discarded. Remember, the fundamental principal of AMQP is that the consumer decides. In this scenario, nobody has bound a queue to the exchange, so the message is discarded because of lack of interest, and in most circumstances, the producer will never know about this.

### Client Design Implications

So from an client application design perspective, you can choose from the following paradigms:

* A producer that never encounters an error when sending a message because they always pre-declare the exchange, but one that is unaware that their messages *may* be silently discarded;
* A producer that is prepared to accept publication failures, because it thinks that this may due to lack of consumer interest.

The upshot of this observation is that Kirk's argument may well be valid if you decide to pre-declare exchanges from the producer's side. However, if you decide that it is not a good idea to send messages down a silent black hole, you may never want to have a producer declare an exchange. This _may_ diffuse the postulate that the producer is coupled to the routing logic.

For completeness' sake, it must be stated that there are a variety of options that could influence the behavior of exchange declaration (e.g. the passive flag on exchange declaration) or sending a message (e.g. the mandatory flag when publishing a message).

[amqp]:http://www.amqp.org/
[kirk]:http://kirkwylie.blogspot.com/2008/07/amqps-semantic-model-and-mismatching.html