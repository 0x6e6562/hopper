Recently I was tracking down a spurious issue in the [RabbitMQ][] Java client library that was returning null values from a call where it just didn't seem possible that it could return null. This is a brief article about a very curious issue in the Java Hotspot compiler. If you are just interested in the HotSpot issue, then just skip the section about it was discovered in the client library.

### The Java Client Bug

When a client application requests a new [AMQP][] channel, the library performs a linear scan for the lowest unused channel number within the current connection. The buggy code looks this:

    public synchronized int allocateChannelNumber(int maxChannels) {
        if (maxChannels == 0) {
            maxChannels = Integer.MAX_VALUE;
        }
        int channelNumber = -1;
        for (int candidate = 1; candidate <= maxChannels; candidate++) {
            if (!_channelMap.containsKey(candidate)) {
                channelNumber = candidate;
                break;
            }
        }
        return channelNumber;
    }

A lot of applications only use a hand full of channels so the lowest used channel number is generally a small value. However, one particular user was creating a new channel for each message they wanted to send, so they began to use a lot of channels. Apart from a linear run time, this shouldn't really be an issue.

As it turned out, after an approximate length of time since the application started, the client library began to return null channels to the application. The culprit was that the allocateChannelNumber() method was returning -1. 

I looked into the offending code and couldn't see anything wrong with it on face value. The symptom looked like a race condition, but the client was only using one thread and the call to allocateChannelNumber() is synchronized as well. Debugging in to where the loop was broken didn't help either - running the code in debug mode never caused the symptom to surface.

The solution to the bug turned to be to either turn of HotSpot compilation or to change the comparison against Integer.MAX\_VALUE to some other value, for example, Integer.MAX\_VALUE - 1.

### The Hotspot Arithmetic

Not really believing that we'd stumbled across a HotSpot bug as opposed to something in our code, producing a stripped down version of the issue seemed to be a good idea:

    public class HotSpotBug {

        public static void main(String[] args) throws Exception {

            int m = Integer.MAX_VALUE;
            int next = 0;
            for (int o = 0; o < 10000; ++o) {

                for (int i = 1; i <= m; ++i) {
                    if (i > next) {
                        next = i;
                        break;
                    }
                }
            }
            System.out.println("Next = " + next);
            return;
        }
    }
  
On face value, you would expect this program to output 10000. If you compile and run this program however, the output on the console varies from invocation to invocation:

    $ java -version
    java version "1.6.0_15"
    Java(TM) SE Runtime Environment (build 1.6.0_15-b03-219)
    Java HotSpot(TM) 64-Bit Server VM (build 14.1-b02-90, mixed mode)
    $ javac HotSpotBug.java
    $ java HotSpotBug
    Next = 354
    
The HotSpot CompileThreshold parameter specifies the number of method invocations/branches before compiling down to native code. If you re-run the program with the HotSpot CompileThreshold set to 0 (which is equivalent to setting -Xint, i.e. forcing pure interpretation), then the program does actually output 10000:

    $ java -XX:CompileThreshold=0 HotSpotBug
    Next = 10000
    
If you play around with the value of the compile threshold, the program outputs different values:
    
    $ java -XX:CompileThreshold=1 HotSpotBug
    Next = 6533
    $ java -XX:CompileThreshold=10 HotSpotBug
    Next = 4986
    $ java -XX:CompileThreshold=100 HotSpotBug
    Next = 2046

### Limitations

I have only run this example on OSX and Linux 64-bit JDKs (both patch level 15 of Java 1.6), so if you run this on a different OS and/or version, YMMV.

### Sun's Bug Database

Some googling on the subject indicates that this is a known issue in the HotSpot compiler. Apparently issue [5091921][bug] was opened in 2004 and has subsequently been marked as having a known cause with a medium priority ([this article][bug_states] explains the bug resolution process for the JDK). In the meantime, don't compare anything to Integer.MAX_VALUE.

### Java Client Fix

The astute read will have noticed that the 0-8 version of the AMQP specification documents the fact that the channel number is an unsigned 16-bit integer. So to an extent, the HotSpot bug masked an issue in the client library that would have arisen once the comparison exceeded 65535.


[bug_states]:http://blogs.sun.com/kto/entry/openjdk_bug_states
[amqp]:http://www.amqp.org/
[rabbitmq]:http://www.rabbitmq.com/
[bug]:http://bugs.sun.com/view_bug.do?bug_id=5091921
