The next major release [RabbitMQ][] contains a feature called QoS, which means egress flow control in [AMQP][] parlance. This article describes a practical application of this feature using a competing consumer work distribution pattern. The example code uses the RabbitMQ Java client, but this feature is not Java-specific and the same principles apply to any language client.

### Basic Qos And Work Distribution

Until now, the basic.qos AMQP command was not implemented in RabbitMQ. This command allows a consumer to choose a prefetch window that specifies the amount of unacknowledged messages it is prepared to receive. By setting the prefetch count to a non-zero value, the broker will not deliver any messages to the consumer that would breach that limit. To move the window forwards, the consumer has to acknowledge the receipt of a message (or a group of messages). By acknowledging a message, the consumer gains credit in the broker which makes it eligible to receive more messages. This credit-based flow control allows the broker to distribute work proportional to the individual processing ability of each worker, as opposed to a simple round robin mechanism.

### The Competing Consumer Example

This [example code][code] contains a main method that demonstrates the mechanics of the basic.qos command in a simple producer/consumer setting. At the time of writing, the current RabbitMQ release version is 1.5.1 and this functionality is only available in the mainline that will be in the next major release. So to run this example, you need at least RabbitMQ 1.6.x or a copy of the latest source tree.

### The Main Test

This is the main method of the example code:

    public static void main(String[] args) throws Exception {
        
        Connection con = new ConnectionFactory().newConnection("localhost");

        Channel throwAwayChannel = con.createChannel();
        String queue = throwAwayChannel.queueDeclare().getQueue();

        ExecutorService threadExecutor = Executors.newFixedThreadPool(5);

        int prefetchCount = 0;

        Worker fast = new Worker(prefetchCount, threadExecutor, 1,
                                 con.createChannel(), queue);
        Worker slow = new Worker(prefetchCount, threadExecutor, 100,
                                 con.createChannel(), queue);

        Producer producer = new Producer(con.createChannel(), queue);
        threadExecutor.execute(producer);

        Thread.sleep(10000);

        threadExecutor.shutdownNow();

        con.close();
        
        System.err.println("Fast worker processed : " + fast.processed);
        System.err.println("Slow worker processed : " + slow.processed);
    }
    
This steps performed in this test are as follows:

* Create a connection to the broker;
* Create a new queue with a server generated name;
* Start a new thread pool to keep the threading readable in this example;
* Set the prefetch count - note that this value is the key variable in the whole example;
* Create two workers:
 * A fast worker that sleeps for 1 cycle every time it receives a message;
 * A slow worker that sleeps for 100 cycles each time;
* Start a producer thread to send stuff to the queue;
* Let the the producers and consumers do their stuff for 10 seconds, after that, shut them down;
* End the connection;
* Print the results.

### The Producer

The code that makes up the producer thread is quite simple and looks like this:

    static class Producer implements Runnable {

        Channel channel;
        String routingKey;

        Producer(Channel c, String r) {
            channel = c;
            routingKey = r;
        }

        public void run() {
            while (true) {
                try {
                    channel.basicPublish("", routingKey,
                                         MessageProperties.BASIC, null);
                    Thread.sleep(10);
                } catch (Exception e) {
                    break;
                }
            }
        }
    }
    
Basically the producer is just sending empty messages directly to the automatically created queue. In order to prevent the producer using all of the available CPU, it is throttled to only send messages every 10 milliseconds.

### The Worker

The worker is a simple extension to the _DefaultConsumer_ class in the RabbitMQ Java library. When the _handleDelivery()_ method is invoked on the main thread, a task is created and submitted to the thread pool, so that the main delivery thread is not blocked:

    static class Worker extends DefaultConsumer {

        String name;
        long sleep;
        Channel channel;
        String queue;
        int processed;
        ExecutorService executorService;

        public Worker(int prefetch, ExecutorService threadExecutor,
                      long s, Channel c, String q) throws Exception {
            super(c);
            sleep = s;
            channel = c;
            queue = q;
            channel.basicQos(prefetch);
            channel.basicConsume(queue, false, this);
            executorService = threadExecutor;
        }

        @Override
        public void handleDelivery(String consumerTag,
                                   Envelope envelope,
                                   AMQP.BasicProperties properties,
                                   byte[] body) throws IOException {
            Runnable task = new VariableLengthTask(this,
                                                   envelope.getDeliveryTag(),
                                                   channel, sleep);
            executorService.submit(task);
        }
    }

The key functional aspect of the worker is that it sets the prefetch window for the channel that it is connecting through. This is achieved by invoking the _basicQos()_ method on the channel object. As indicated beforehand, this is the key setting for the entire example.

### Submitting Tasks

By sleeping for a variable length of time before the message is acknowledged, the _VariableLengthTask_ simulates the execution of jobs that require different amounts of time to complete:

    static class VariableLengthTask implements Runnable {

        long tag;
        long sleep;
        Channel chan;
        Worker worker;

        VariableLengthTask(Worker w, long t, Channel c, long s) {
            worker = w;
            tag = t;
            chan = c;
            sleep = s;
        }

        public void run() {
            try {
                Thread.sleep(sleep * 10);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }

            if (chan.isOpen()) {
                try {
                    chan.basicAck(tag, false);
                    worker.processed++;
                } catch (IOException e) {}
            }
        }
    }

After each acknowledgement, the counter for the amount of messages processed is incremented. This value will be used later on to compare the processing capacity of the two workers.

### The Initial Test

The initial version of the test sets the _prefetchCount_ variable to zero. Semantically, this is the same as not setting any prefetch count at all, or in other words, not using any egress flow control. When the test was run for 10 seconds with a prefetch of zero, the following throughput results were achieved:

    Fast thread processed : 40
    Slow thread processed : 36
    
At first glance, this seems a little counterintuitive - how can it be that both workers processed about the same amount of messages, although the processing ability of the fast worker is 100 times that of the slow worker?

The answer lies in the lack of flow control. When the broker is not instructed to perform flow control, it uses a simple round robin mechanism to deliver messages to consumers. The upshot of this is that the fast worker is starved of tasks and is effectively throttled to the speed of the slow worker.

### Releasing The Handbrake

Setting the prefetch count to a non-zero value allows the broker to perform credit based flow control between each consumer and hence achieves a *fairer* distribution of work. Using a prefetch count of one yields these results:

    Fast worker processed : 945
    Slow worker processed : 9

In this scenario, the respective amount of messages processed by each worker is roughly proportion the relative amount of time it takes each one to complete a task. In this example, the slow worker sleeps for 100 cycles per message whilst the fast worker sleeps for only 1 cycle and the difference in the total amount of work done is approximately a factor of 100. The net result is that with flow control, the same amount of processing resources were able to complete 954 tasks as opposed to the case without flow control, where only 76 tasks were processed.

By using flow control, the broker is able to distribute load in a more fine grained fashion, resulting in the most performant workers receiving the bulk of the work.

The advantage of this is that an unexpected slow down by one worker will not affect the overall ability of the worker pool to perform.

### Considerations

Whilst egress flow control can be very useful in tuning your processing pipeline, it must be pointed out that maintaining the ack-based credits on a per-channel basis incurs some overhead that slows down the overall throughput rate. This is because the broker has to carry out a lot of accounting in order to manage the message flow. In some cases, a reduction in throughput of about 15% has been observed. Your mileage may vary.

[code]:/storage/code/QosDemo.java
[amqp]:http://www.amqp.org
[rabbitmq]:http://www.rabbitmq.com