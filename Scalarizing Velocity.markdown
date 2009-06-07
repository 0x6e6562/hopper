It is quite well documented that [Scala][] is a boon to those who need to extend or integrate Java based applications and libraries. Recently, I found myself implementing some some new back end functionality for a website that uses [Velocity][] to render its frontend. Whilst doing this, a few gotchas arose:

* Scala collections do not work out of the box with a Velocity foreach loop;
* Velocity expects the attributes of an object instance to be exposed via the JavaBean getter/setter convention;

However, a small amount of googling some simple solutions for these problems.

### Automatically Convert Scala Collections To Java

Whilst the reverse operation is well documented, there does not appear to be an inbuilt mechanism to convert Scala collections to Java. The solution to this has been provided by Jorge Ortiz's [scala-javautils][] library on Github.

This uses an implicit conversion that allows you to invoke a function called _asJava_ on any Scala collection to get it converted to a Java collection. Just refer to the README on the Github project page to see how easy this is. After applying this to my Scala collections, the Velocity templates are now able to iterate through them.

### Expose Attributes As JavaBean accessors

One of the nice things about Scala is the paradigm shift away from unnecessary JavaBean accessors. However, Velocity, unless I'm unaware of of some magical configuration parameter, follows the JavaBean mantra, which can be a hassle when you just want your template to read the properties of a class instance.

Thankfully there is an on-board feature that can auto-generate getters and setters for your attributes. By annotating a field with the @BeanProperty annotation, the Scala compiler will automatically generate these methods for you, thus saving you from unnecessary boilerplate in your Scala code. An example looks like this:

    package foo
    
    import reflect.BeanProperty
    
    class Bar {
      @BeanProperty var id:Long = -1
    }
    
This allows you to use plain jane JavaBean accessors in your Velocity templates.

### If Scala Can Drive Velocity

A third option may be of interest to you, if you are invoking the template application from within Scala code. Martin Kneissl wrote this [Scala wrapper for Velocity][scala-velocity], which addresses the Java collection issue in Scala.

YMMV.

[scala-javautils]:http://github.com/jorgeortiz85/scala-javautils/tree/master
[scala]:http://www.scala-lang.org/
[velocity]:http://velocity.apache.org/
[scala-velocity]:http://www.familie-kneissl.org/open-source/scala-velocity