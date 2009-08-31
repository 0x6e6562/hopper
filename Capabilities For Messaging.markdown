This article introduces the potential usage of capabilities as an access control mechanism in a messaging broker. It describes the motivation behind a proof-of-concept implementation that demonstrates the feasibility of a capability based approach to securing access in the [RabbitMQ][] message broker. Whilst it was decided not to further develop this paradigm, the experiences gained from it may be valuable for anybody contemplating using capabilities to implement access control. Though not strictly necessary, a basic understanding of the [AMQP][] execution model can be advantageous to appreciate the implications of this article.

### Introduction

The RabbitMQ community has frequently requested the core engineering team to deliver some kind of access control mechanism that allows untrusted users to
connect to a broker and send or receive messages in a fashion that a server administrator can control. Classically, this kind of functionality falls under the realm of [access control lists][acl], or ACLs for short. The only problem the remained was to categorize the types of operations that an administrator would typically want to control.

This appeared to be a relatively straight forward requirement for which something quite simple could be developed. In a pioneering spirit, the engineering team asked itself if using traditional ACLs was the best (or only) solution to the problem. In order to answer this question, they searched for an alternative approach to solving the same problem.

After researching the topic, capabilities were identified as a potential alternative to ACLs. The pros and contras of both approaches were deliberated as well as implementing proof-of-concepts and in the end, the engineering team decided to go for ACLs after all. This was due to the changes to client side libraries that would have to take place in the case of capabilities. The ACL approach was not afflicted by this. Despite capabilities being ruled out for RabbitMQ, it remains a simple and powerful concept and the experiences gained from this exercise deserve some kind of documentation.

### Capabilities

A [capability][] is an unforgeable reference (or key) that allows access to an arbitrary entity. This follows the principle of least authority whereby a resource is accessed using the smallest possible set of privileges.

Imagine you want to prevent access to the front door of a house. Using a ACL, you would first identify the person wishing to enter the door and then verify that this person is actually authorized to open the door.

Using a capability, that person would be given a key to unlock the door. The mere possession of the key enables that person access to the house. In this form of access control, it is irrelevant who wants to enter, as long as they have the right key.

### Decentralizing Access Control In RabbitMQ

The goal of this article is not to explain in full or discuss the relative merits of ACLs contra capabilities - there is enough academic literature [covering this topic][myths]. Moreover, the goal of this article is to illustrate the benefits that a capability approach may have brought to a messaging broker.

Capabilities follow the philosophy that possession is nine tenths of the law. If I create something, therefore I own it. For example, if I create a queue in RabbitMQ, the server can generate a cryptographically secure name for my queue that nobody could guess. As long as I keep this name secret, nobody can feasibly put stuff into my queue or take stuff from it.

If I wanted to grant somebody access to this queue, then all I have to do is to give them a copy of the name. Once they have this name, they can do whatever they want with the queue, because they then possess a _reference to the underlying entity_ that they want to access. If they can see it, they can access it. If they can't see it, they can't access it.

The upshot of this is that I can delegate authority to whoever I like without consulting the administrator. This decentralization potential brings huge gains - one no longer has to administer access to the message broker - whoever creates an entity controls the access to it. From a practical perspective, this
 
* frees a server administrator from having to care about access control at all;
* allows controllable access to unknown entities, e.g. people outside of a particular organization;

### Delegating And Revoking Privileges

The second main advantage of capabilities is the ability to create delegates that encapsulate the access to the capability to access the actual underlying resource.

If I give somebody a key to a door, then they can open it. I can't take the key back, because it would useless if they had made a copy of it in the meantime. However, if I put a second door with an open passageway in front of the first, I can give a person a key to the second door, behind which a concierge of my appointment has the key to the first, they can access the first door using the key of the second. If I decide at some later stage that I no longer that person, I can just remove the concierge with the key to the first door. Although I cannot take back the key to the second door, by removing the concierge, I render the second door useless and hence prevent access to the first. 

This scheme can be extended recursively. If the person I want to authorize wants to authorize a third person in the same way, then that's their prerogative. Whether the second person creates their own concierge or not, by removing my concierge I prevent access to both of them. In this fashion the system is always recursively rooted so that you can safely give out as many capabilities as you want, provided that the root capability is keep secret.

The benefit of this in a multi-user and multi-participant messaging fabric is that it no longer requires a central instance to broaden or narrow the scope of access of any entity you would like to protect.

### The User Decides How Access Is Granted

In contrast to an ACL based system, where the intentions of a user have to be taxonomized in order for the system to compute access, a capability based system leaves this up to whoever possesses the capability to access a resource.

In the RabbitMQ scenario, I may create an exchange where I want to give certain people read access (to bind queues to the exchange) and certain people write access (in order to publish messages). Using an ACL model, the server implementor would need to identify these two usage patterns a priori. Using a capability model, whoever creates the exchange can create separate delegates to allow read access and write access. In the concierge analogy above, this is akin to employing two concierges - one to let people in and one to let people out.

This approach allows the user to define their own access semantics, so not only do they control who has access to what, they can also determine what access to a particular resource actually means to them.

### How This All Works

In order to be able to hand out access and take it back at some later stage, a resource is typically protected by two delegates. These two delegates form the fundamental facets of the delegation and revocation mechanism:

* The forwarding facet is a delegate whose capability is given to the person that you want to delegate something to;
* The revoking facet is the delegate who the forwarding facet points to and in turn can eliminate the capability of the forwarding facet.

The forwarding facet is given out the delegatee, whilst the revoking facet is retained by the delegator. The delegatee can access the underlying resource as long as the delegator has not exercised the revoking facet. This way the delegator does not need to change the locks of the real entity they are trying to protect.

Furthermore, since all access control is hierarchically delegatable, the whole hierarchy has to be rooted by somebody. This is analogous to the root user on a Unix operating system.

### The Proof Of Concept

The proof of concept code can be downloaded by the [RabbitMQ Mercurial repository][hg], either using the hg command line tool or downloading a tarball. You will a recent version of Erlang on your system to run this. These instructions are based on checking out the source from hg:

    $ hg clone http://hg.rabbitmq.com/rabbitmq-codegen
    $ hg clone http://hg.rabbitmq.com/rabbitmq-server
    $ hg up -C bug20149
    $ make
    
After this has successfully been built, start an Erlang shell and run the main test:

    $ erl -pa ebin
    xlr8:rabbitmq-server 0x6e6562$ erl -pa ebin/
    Erlang (BEAM) emulator version 5.6.5 [source] [smp:2] [async-threads:0]
    Eshell V5.6.5  (abort with ^G)
    1> rabbit_capability:test().
    ok
    2> 

This runs two different tests:

* exchange\_declare\_test;
* bogus\_intent\_test

which will be explained separately.

These tests use a simulation of the core channel API in the RabbitMQ broker. Whilst this not the exact API, the observant reader will recognize the similarity. The point is that the code serves as a proof of concept, that may have fully integrated in the server codebase, had ACLs not been preferred over capabilities.

#### Restrictions and Assumptions

The astute reader of the code will notice that the command set of AMQP has been marginally tweaked to allow for a straight forward implementation of capabilities. The rationale behind this is twofold:

* The current AMQP specification is 0.9.1 and will be revised before the protocol goes 1.0 - this gives an opportunity to extend the command set so that it can _carry_ the necessary capabilities between client and server;
* A client library would have to be modified in any case in order to provide key distribution and delegate creation.

Hence it was decided to modify the execution model slightly in order to to prove this concept. It is possible to avoid changes to the execution model, but doing so provides clarity to the intent of capabilities.

### Exchange Declare Test

This is a test case to for creating and revoking forwarding capabilites, which follows the following steps:

1. There is a root capability to create exchanges;

2. Root creates a delegate to this functionality and gives the forwarding facet to Alice;

3. Alice now has the capability C to a delegate that can execute the exchange.declare command. To declare an exchange, Alice does the following:

 * Sends an exchange.declare command as she would in a world without capabilities with the exception that she adds the capability C as an argument to the command;
 * The channel detects the presence of the capability argument, resolves the delegate function and executes it with the exchange.declare command from Alice in situ;
 * The result is returned to Alice;
 
4. If Alice wants to delegate the ability to create exchanges to Bob, she can either:

 * Create a delegate that forwards to the delegate for which Alice
 * has the capability C;
 * Just give Bob the capability C;

### Bogus Intent Test

This is a test case to for creating and forwarding capabilities on the same exchange entity. This demonstrates how different delegates encapsulate different intents in a way that is specified by the owner of the underlying entity:

1. There is a root capability to create exchanges and bindings as well as to publish messages;

2. Root creates a delegate to these functionalities and gives the forwarding facets to Alice;

3. Alice creates an exchange that she would like to protect;

4. Alice creates a delegate to allow Bob to bind queues to her exchange and a delegate to allow Carol to publish messages to her exchange;

5. After this has been verified, Bob and Carol try to be sneaky with the delegates they have been given. Each one of them tries to misuse the capability to perform a different action to the delegate they possess, i.e. Bob tries to send a message whilst Carol tries to bind a queue to the exchange - they both find out that their respective capabilities have been bound by intent.

### Observations

The proof-of-concept code is small and concise - most of the code is test code or code designed to set up the root capabilities to system. The actual capability checking code is minimal. Whilst it does not use the real API directly, the proof-of-concept API is so similar to the real API that a conversion would be trivial. Because of the changes to the client, it was decided that ACLs were a more appropriate choice for an access control mechanism.

[amqp]:http://www.amqp.org
[hg]:http://hg.rabbitmq.com
[myths]:http://srl.cs.jhu.edu/pubs/SRL2003-02.pdf 
[rabbitmq]:http://www.rabbitmq.com
[acl]:http://en.wikipedia.org/wiki/Access_control_list
[capability]:http://en.wikipedia.org/wiki/Capability-based_security