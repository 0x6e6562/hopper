Recently, I ran into a situation where I needed to supply a Scala component with a lambda that took a externally defined marker interface as an argument. In the particular scenario, I wanted to pass in an implementation of the marker, but the Scala compiler didn't allow it. This article summarizes the scenario and illustrates a simple workaround. Whether or not this example demonstrates the best possible factoring for the scenario remains an open question. 

### Scenario

Imagine that you have a simple marker trait in Scala (which would be a marker interface if you wrote it in Java):
	
	trait Marker

To illustrate this problem, we will need a trivial implementation of the Marker trait:
	
	class Implementation(val text:String) extends Marker

We we will define a simple consuming application that is instantiated with a lambda that takes the marker trait as an argument. This particular example uses the function in order to produce print statements:

	class Consumer(val fun:Marker => String) {
	  def print(marker:Marker) = println(fun(marker))
	}

Next, we'll need a simple test case to wire all of the components together and run a Hello World scenario:

	object Test {
	  def main(args: Array[String]) = {
		var impl = new Implementation("Hello World!")
		var fun : Marker => String = { i:Implementation => i.text }
		val consumer = new Consumer(fun)
		consumer.print(impl)
	  }  
	}

When you try to compile this, you will run into the following error:

	Information:Compilation completed with 1 error and 0 warnings
	Information:1 error
	Information:0 warnings
	C:\Users\0x6e6562\Workspace\test\untitled\src\test\Test.scala
		Error:Error:line (4)error: type mismatch;
	found   : (Implementation) => String
	required: (Marker) => String
	var e : Marker => String = { i:Implementation => i.text }

The solution to this is to turn the function on the fourth line into a partial function using a case clause:

	var fun : Marker => String = { case i:Implementation => i.text }
		
This is effectively doing the same thing as an explicit type cast:

	var fun : Marker => String = { i:Marker => i.asInstanceOf[Implementation].text }
		
### Discussion

Obviously the Consumer class would throw an exception if you passed in something other than an instance of the class called Implementation. Furthermore, it is question as to whether the code is correctly factored, considering that it possible to invoke the function in a non-typesafe fashion.

For anybody interested in reading how the partial function is constructed, [this article about case statements][partial] may be of interest.

[partial]:http://jim-mcbeath.blogspot.com/2009/10/scala-case-statements-as-partial.html