One of the items that the [RabbitMQ][] team has been concentrating on recently, is a major piece of refactoring that intends to achieve linear scalability of routing within the message broker. Actually doing the refactoring work turned out to be a lot less effort than designing and implementing a strategy to verify that the resulting patch set did indeed scale linearly. This is an article about the approach that was taken in order to measure the scalability of the system. It is assumed that the reader is familiar with the execution model of [AMQP][].

### Problem Statement

Reaching an objective definition of the scalability of a system is not always an cut and dried task. In many cases it depends on what characteristics you consider to be relevant for measuring scalability.

In the case of an AMQP message broker, one could simply consider using either the message throughput or the end to end latency as a key performance indicator. A simple test could start recording the per-message cost and begin to increase the volume of messages submitted to the system to see how the average cost behaves.

This sounds plausible, but where does one draw the line? Do both dimensions, throughput and latency, have to exhibit the same orders of magnitude with respect to cost? Would it be acceptable to trade off better performance in one dimension to the detriment of the other? And if so, how do you make this decision? Is it based on the use cases of the messaging system that you are _currently_ aware of?

Without elaborating on all of the factors that one could feasibly consider, the overall approach of this routing scalability patch intends to make RabbitMQ exhibit no worse than linear costs for creating and routing to an arbitrarily large number or queues. Consider for example, a system in which millions of queues each need to receive certain messages. Not only does the exchange routing have to be able to compute routes in the face of an extremely large set of queue endpoints, but more importantly, the broker first needs to be able to create these queue entities in linear time before it can even think about sending messages to them.

Therefore, the central requirements of this development activity were defined thusly:

* Creating and deleting queues and bindings between exchanges and queues should be executed in no worse than linear time for an arbitrarily large amount of queues and bindings per queue;

* The per message cost of routing a message to an arbitrarily large number of bound queues should be no worse than linear.

The chief motivation of this patch set was the observation that these desired system characteristics were not delivered by the 1.4.0 release of RabbitMQ.

### The Patch Set

The basis of comparison for this article is a scalability test that takes a ceteris paribus approach between the previous release of the RabbitMQ broker (1.4.0), and a patch set in which the routing mechanism of the broker was fundamentally changed. The patch set has the code name 18776, which is an internal bug id for the RabbitMQ team. For here on in, the 1.4.0 release will be referred to as the default branch, and the patch set as the 18776 branch.

### Running The Test

The test that implements the verification of scalability is now part of the test suite that ships with the forthcoming release of the RabbitMQ Java client, and is currently available in the RabbitMQ source tree for those who want to run it for themselves.

The test has been designed to operate in an exponential fashion in order to highlight the cost of transitioning from a one order of magnitude to another. By choosing an exponential strategy, it becomes easy to identify super-linear runtime complexities when the points are plotted to a logarithmic scale.

The test class, called ScalabilityTest, was executed using the following command line arguments:

    java -cp $CP ScalabilityTest -b 2 -x 15 -y 11 -f LABEL
    
The meaning of these parameters are as follows:

* b - the base upon which each order of magnitude is computed, so for b = 2 this will be 1, 2, 4, 8, 16, 32, 64, 128, 256, 512, ...... , 2 ^ (x+y) / 2;
* x - the exponent for the amount of queues to create, i.e. 2 ^ 15;
* y - the exponent for the amount of bindings per queue to create, i.e. 2 ^ 11;
* f - this is the label of the time individual series for plotting purposes;

Because of the exponential nature of the test, it is necessary to limit extreme orders of magnitude on both axis's at the same time. If one did not do so, running the test for 2 ^ 15 queues and 2 ^ 11 bindings per queue at the same time would take a prohibitively long time to execute. To avoid this, each iteration of the benchmark is limited by the maximum exponent of each axis, i.e. (x+y) / 2. The effect of this is that the plotted results will form a triangle along this average.

To get a full description of all of the possible command line arguments, you can run:

    com.rabbitmq.client.test.performance.ScalabilityTest --help

### Interpreting The Results

It must be pointed out that this test is designed to verify the overall runtime complexity of the routing functionality in Rabbit with respect to the efficiency of binding creation, deletion and the routing of messages. The results that I am presenting are indicative of the _orders of magnitude_ that one can reasonably expect from a default RabbitMQ installation on commodity hardware. _For the record_, these tests were deliberately run on a low powered old desktop machine. Hence, the results _should_ not be interpreted as the absolute performance characteristics of RabbitMQ on appropriately resourced hardware.

The following graphs have been plotted against logarithmic scales. This is because the test case uses exponents to define the amount of queues and bindings per queue that should be created for every level of the test.

The units of the graph are in microseconds, but as indicated above, one should not read too much into the absolute value of these particular results. Your mileage may vary.

### Queue/Binding Creation And Deletion

On the deletion and creation graphs, the x axis depicts the number of queues, whilst each series represents different bindings per queue of increasing powers of 2, i.e. 1, 2, 4,... 2 ^ (x+y) / 2.

The y axis shows the average time per operation, i.e. the creation and deletion of a single queue together with it all of its bindings.

The first graph shows the comparison between the 1.4.0 version of RabbitMQ (the default branch) and the routing patch (branch 18776):

![][creation]

On the default branch, the cost of queue creation increases at a worse-than-linear rate as the number of bindings per queue increases. On the 18776 branch, however, the increase is about linear.

The second plot depicts a comparison between the two versions of RabbitMQ for the time is taken to delete the queues and bindings created in the first step:

![][deletion]

The deletion chart shows similar runtime patterns to the creation chart, but it is not symmetric to the creation side. On both the default branch and the 18776 branch, the cost of queue deletion increases at a worse-than-linear rate as the number of bindings per queue increases. It's not quadratic, probably something in the order of n log n. The base cost of deletion is much higher
on the 18776 branch, but it is probably acceptable for the upcoming release.

In summary, the cost of queue and binding creation and deletion on the default branch increases significantly once the test is extended beyond 100 queues. By contrast, on the 18776 branch, the cost actually decreases slightly. This was the main objective of the 18776 patch set.

### Message Routing

On the routing graphs, the x axis characterizes the number of bindings per queue. Each line represents a different number queues of increasing power of 2. The y axis is the average time it takes to route a single message to each bound queue.

In analogy to the charts for queue creation and deletion, the two series on this plot represent the 1.4.0 release and the 18776 patch set, respectively:

![][routing]

This indicates that the routing performance on both the default and the 18776 branches are close to constant regardless of the number of bindings. It appears to increase linearly with the number of queues. The base cost of routing is higher on the 18776 branch, but this can presumably be reduced by introducing route caching into a subsequent release of RabbitMQ. Retaining the same runtime complexity (though not necessarily same base level) of routing performance was the secondary objective of this patch.

### Conclusion

Overall these results demonstrate that we have achieved the objectives of this
patch and have not broken anything any other aspect of the broker.

### Acknowledgements

I would like to say thank you to Matthias Radestock specifically for the hard work that he has put in to verify the verification process and to help draw conclusions from the data that was collected. Without that, this article would not have been possible.

[rabbitmq]:http://www.rabbitmq.com
[creation]:/storage/diagrams/18776-default-creation.png
[deletion]:/storage/diagrams/18776-default-deletion.png
[routing]:/storage/diagrams/18776-default-routing.png
[amqp]:http://www.amqp.org