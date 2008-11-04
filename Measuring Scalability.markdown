One of the items that the RabbitMQ team has been concentrating on recently is
a major piece of refactoring intending to acheive linear scabaility of routing
within the message broker. Actually doing the refactoring work turned out to
be a lot less effort than designing and implementing a strategy to verify that
the resulting patch dee indeed scale linearly. This is an article about the
approach that was taken to measuring the scalablity of the system.

### Problem Statement

S RabbitMQ Java client with the

### Running The Test

The ScalabilityTest class was executed using the following command line
arguments:

    com.rabbitmq.client.test.performance.ScalabilityTest \
    -b 2 -x15 -y 11 -f LABEL_FOR THE SERIES-

### Interpreting The Results

![][creation]

[creation]:/storage/diagrams/18776-default-creation.png
[deletion]:/storage/diagrams/18776-default-deletion.png
[routing]:/storage/diagrams/18776-default-routing.png
