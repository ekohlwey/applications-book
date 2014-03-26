Exception Handling
==================

Effective exception handling is fundamental to good application
design. While there is much debate about what exception handling
methodology to use, there is no debate that one should decide
on and use a uniform methodology within one's application.

Without good exception handling, it is likely that your application
will loose or corrupt data. Good exception handling also makes operating your
application easier because the folks installing your application
will have an easier time figuring out what has gone wrong and fixing
it (something always does).

Below I outline _my_ preferred methodology, but hope that you will
take the time to learn about, and appreciate other methodologies.
If you don't use them, you will almost certainly happen across them
when using other folk's libraries and need to understand how to
use them effectively within your application.

Overview
--------

The fundamental assertion of this chapter is that good exceptional
control flow is both _precise_ and _checked_. _Precise_ exception
handling means that it is clear exactly what application state the 
application architect meant to convey by throwing the exception.
_Checked_ exceptions are any subclass of `java.lang.Throwable` that
do not inherit from `java.lang.RuntimeException`. Usually this
means that the exception is a subclass of `java.lang.Exception`. 
I will not delve too far into the design or rationalizations of Java's
exceptional control flow system - if you Google on the topic
you will find a myriad of official documentation and blog posts that
engage in deep discussion.

I like to think of a checked exception as an alternative return type of
a method. More on this later.

Precision
--------- 

Precision means that you understand exactly what the application developer
intended to convey when the exception was thrown. An important aspect
of precision is that an exception type (by type I mean the Java class that
implements the exception) should be unique to the application that throws the
exception, and should not be an exception thrown by a library.

Take, for example, a theoretical application that uses ZooKeeper to
store a list of Apples. Below is a common novice solution to handling
a `ZookeeperException` (which ZooKeeper uses to convey a problem back to API users):

```java
class AppleManager {
  
  public List<Apple> getApples() throws ZookeeperException {...}

  public List<Apple> setApple(int i, Apple apple) throws ZookeeperException {...}

}
```

The method signature above simply propagates the `ZookeeperException` to... well
wherever. What this typically means is that the programmer who wrote the code
had no idea what to do, so they just sent it back to the caller.

This is a bad idea. Consider the following:
- What happens if I no longer wish to use Zookeeper to store this information?
- Have I considered that Zookeeper is using precise exception handling, and that
  I am actually expected to re-try certain requests because it is part of the
  Zookeeper protocol?

For both of these reasons, the following definition would be more appropriate:

```java
class AppleManager {
  
  public List<Apple> getApples() throws UnableToContactDataStoreException {...}

  public List<Apple> setApple(int i, Apple apple) throws UnableToContactDataStoreException {...}
}

```

I don't really care that the application can't contact Zookeeper. I care that the
underlying data store is having problems and I need to tell someone to go look at it
and fix it. Furthermore, as indicated by the aforementioned considerations, many
subclasses of ZookeeperException are intended to be caught at the location of the method invocation.
(as a side note, this behavior, which many find counterintuitive, was a major motivator
for the creation of the Curator project, which most applications use instead of Zookeeper).

If I am writing a user interface using this API, I can catch `UnableToContactDataStoreException`
and present the user with a helpful message (for most web services, that would be to
wait for a moment; an administrator has been notified and the service will
be back online shortly), or notify an administrator that there is a problem.

### Error Data

Often, exceptions occur because of badly-formatted data. In such a case, I like to provide
the example datum on the exception because inspecting error data is such a common task, even
if I don't happen to inspect the error datum at the time I have written the class.

So, for example, I might have the following exception definition:

```java
public class JetDeserializationException extends Exception{

  public JetDeserializationException(String jetRecord){
    super("Unable to deserialize jet " + jetRecord);
  }

}
```

This is ok since it is precise, but if I happen upon a use case where I want to look
at what happened to the jet record that prevented me from parsing it, I need to
go back and refactor all the places where this exception is created, which is inconvenient.
If I just designed the exception to provide an example datum to begin with (which
is a very common need) then I wouldn't have to muck around with refactoring later.

Here's a better version of the same class, which contains all the information I need to highlight
the `Jet` serialization error in a GUI (as an example):

```java
public class JetDeserializationException extends Exception{
  ...
  public JetDeserializationException(String jetRecord, int offset, char got, char[] expected){
    super("Unable to deserialize jet " + jetRecord + " at " + offset + " got " + got + " expected one " +
        "of " + asList(expected) );
    this.record = jetRecord;
    this.offset = offset;
    this.got = got;
    this.expected = expected;
  }
}
```


Checking
--------

Checked exceptions simply mean that the compiler will fail to compile your code if you
do not explicitly do something with an exception, either by propagating it, handling it,
or swallowing it.

The intent of this pattern was to force Java developers to actually do something meaningful
with exceptions. Unfortunately good exception handling practices are not taught in many
curricula and good practices are not usually intuitive. There is still significant debate
as to weather or not Java should feature runtime, checked, or both types of exceptions; I
think we can safely assume that the debate will never be settled and left to application
architects to resolve on a case-by-case basis.

You can think of a Java checked exception as an additional return type for a method. In
fact, some frameworks (such as Jersey, a REST framework) explicitly use exceptions for
this purpose (aside: note that Jersey actually chose to use runtime exceptions, even
though the exceptions a method throws are really a "special" return type for the
REST method).

Consider the following example:

```java

class JetLauncher {

  public void launchJet(Jet jet) throws JetLaunchException { ... }

}

class JetCLI {

  public void main(String[] args){
    ...
    try{
      jetLauncher.launchJet(jet);
    } catch (JetLaunchException e) {
      if(e.jetRunwayNotConfigured()){
        System.out.println("You must configure the runway");
      } else {
        System.out.println("Unable to launch jet for an unknown" +
            " reason. See the below stack trace:");
        e.getCause().printStackTrace();
      }
      System.exit(-1);
    }
  }
}

@Path("/jets")
class JetResource {

  ...
  
  @Path("launch/{name}") 
  public void launch(@Param("name") String name){
    ...
    try{
      jetLauncher.launchJet(jet);
    } catch (JetLaunchException e) {
      // send an e-mail to the system admin that the jet 
      // didn't launch -- come check it out!
      ...
      // throw a WebApplicationException with a 500 
      // status and a helpful message as the content.
      ...
    }
  }

}

```

The class `JetCLI` and `JetResource` both need to do different things if they encounter a 
`JetLaunchException`.

Also, notice that in JetCLI we've actually looked exactly at the exception that has occurred in
order to provide more directed feedback to the user. Likewise, `WebApplicationException` has
a constructor which takes a `Response` object, which can contain important return information
such as a return code and a new content body, representity by the `entity(Object entity)` method
of the response builder.

Retries and IOExceptions
-----------------------

### What IOException Means, and What You Should Do

`java.io.IOException` is probably the most frequently abused and misunderstood exception in the
entire world. Misunderstandings of its nuances have lead to the creation of entire exceptional
control flow strategies which I find dubious.

Its meaning is simple, nuanced, vague, and complex (its like a poem :)).

Basically, an `IOException` (IOE for brevity) means that something happened with 
an IO device that prevented the request from working, that you should think 
about the context in which the error happened, and maybe retry the request in a
 few moments before failing entirely.

Most people don't know that IOE's should be retried in many contexts. To know when,
you must think about your application, what it has asked for, and weather or not it makes
sense to ask again. Consider the following:

- Could the error have been caused by someone restarting a server?
- Is the service that I'm calling highly available, or should I try back?
- Can I reasonably try back in a few milliseconds and not effect application performance?
- Could the error have been caused by someone plugging and unplugging network cables?
- Am I using files on the hard drive to implement some sort of work queue or log?

Some frameworks, such as Zookeeper, have their own exception handling strategies that are similar-
the checked exception is being presented to you specifically so that you can re-try the request.
Be aware of this fact and read the documentation of distributed application libraries with
more scrutiny. Also, be aware that you may use this strategy in your own application.

Writing code to retry things can also be tedious. You can envision having this whole for loop
where you have to call `Thread.sleep()` and then mark time, maybe do some math to make the second
retry longer, etc. etc. What a hassle!

Luckily there's a library that someone has written, which is an extension of the Guava libraries
from Google, that you can use to retry lots of exception types (not just
IOE). You can find it on [rholder's Github](https://github.com/rholder/guava-retrying), or check
the thread on an 
[offical request to integrate the functionality](https://code.google.com/p/guava-libraries/issues/detail?id=490)
directly into Guava for some alternatives, and updates on when it can be expected in the main 
Guava release.

### IOException Abuse

IOE is so common in method signatures that many folks also feel it is an invitation to
get rid of checked exception that they don't understand or have no meaningful way to report.
Countless thought clouds have risen above programmers heads: "Hmmm, what should I do with that
Exception... I don't know, but I'm implementing a method that can throw an IOE... I know! I'll wrap 
that inconvenient exception and throw it over the fence!". Of course, that is a bad practice
because there's no problem with the network, disk, file system contents, etc. **Often, this
line of thinking is a good indication that you need to rethink your application design, usually
around the return types or error handling that you utilize in an RPC interface**.

Interruption
------------

Interruption in Java has a long and storied history. Without rehashing it, this is what you need to know:

- `InterruptedException` exists so that operating system libraries have a way to notify sleeping
  threads that something has happened that they need to pay attention to (perhaps populating a
  queue with mouse click events or something).
- Most people think the introduction of `InterruptedException` was a mistake
- Some applications use it to do certain concurrency tasks
  and unfortunately, for this reason, we can't get rid of it.
- Although its unlikely that you will ever introduce a bug into your code by ignoring an `InterruptedException`,
  you will probably never figure it out if you do.
- You should always follow the following recipe for handling an `InterruptedException` that you don't know
  what to do with

```java
  ...
  } catch (InterruptedException e){
    Thread.currentThread().interrupt();
  }
```

The reason for this is:

1. Catching the exception clears a flag on the thread that was interrupted. This flag is used for communication.
2. Setting the flag again ensures that other frameworks can check the flag if they need to.

There are many locking constructs in `java.util.concurrent` that you can use instead of `Object.wait()` if you
want these details to be handled transparently by some library code. They also provide clearer locking
semantics and a number of implementations of useful lock types.
 
