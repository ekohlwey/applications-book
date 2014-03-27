Dependency Injection and Guice
==============================

Guice is a dependency injection framework for Java. You may recognize some of
the concepts from similar frameworks such as Spring. Although conceptually
similar, Guice is completely different.

Guice has a ton of [great introductory material
here](https://code.google.com/p/google-guice/wiki/GettingStarted), so I won't
re-hash it. The material presented below is supplementary to the stuff they
talk about.

Effective DI Design
-------------------

Effective DI in Java is a combination of functional style and strongly
object-oriented design.  Perhaps most importantly, good DI discards many
practices common to most imperative programmers' styles.

I highly recommend that, in addition to the brief overview below, you take the
time to read _Design Patterns: Elements of Reusable Object-Oriented Software_.

### Names in Java

Early in my career, people often told me that Java was self-documenting. I
didn't really understand that statement for a long time. It didn't really seem
like Java had any more or fewer documentation features than most languages. I
read (or viewed) essays such as [the kingdom of
nouns](http://steve-yegge.blogspot.com/2006/03/execution-in-kingdom-of-nouns.html)
and [recovery from
addiction](http://ia600209.us.archive.org/14/items/SeanKellyRecoveryfromAddiction/Recovery_from_Addiction.mov)
that were highly critical of Java's language features and, at the time, agreed.

I worked on several projects based on Python and Ruby, and quickly found that I
was often confused by applications that people had developed. The lack of
explicit naming of types (or lack of real typing at all) in these languages
created tremendous confusion when working within a larger development team.

Today I primarily use Java to write REST services that are maintained by teams
of 7 or more people.  Such applications cannot realistically be written and
maintained in many other languages because it is not evident why a code module
exists, or what it does, and nobody ever takes the time to comment their code.

In Java, everything has a name, possibly several names, to reinforce why the
class exists and what it does.  Java has mind-melded with IDE's such as Eclipse
to provide effective tools to navigate and understand a code base.

Java works really well when all of your verbs have one method, and all of your
messages and state basically have only getters and setters. This simplistic way
of writing Java makes it very easy to create simple, understandable
applications with fewer warts.

#### Named Tuples

One of the features that I liked initially about most functional and dynamic
languages was the tuple.  It is a handy way to put things together that belong
together, or to basically create a type.

An example of a tuple in an ML-derivative might look as simple as this: 

```ml
("M-71",4500,500) -> (string,int,int) 
```

I've created a type of `(string,int,in)`, but folks who attempt to work on my
application in the future will have no idea why these things belong together or
what they mean. I must trace through my application and inspect function after
function to understand why these things are associated together, theorizing and
testing my suspicions about what the original programmer intended (because of
course, for brevity, everything has a short name and there's no comments!).

Bad Java programmers who like tuples might do something really ugly, like this
(and then complain about Java's lack of a good typed tuple construct):

```java
Object[] jet = new Object[]{"M-71",4500,500};
```

This is sort of OK, because the variable name hints to us what the object is,
but we loose a lot of information about its "fields" and also loose even the
variable name hint outside of the context of the declaration, when this object
is passed into other methods, etc.
 
A similar declaration in Java is large but very expressive:

```java
public class Jet {
  private final String callsign;
  private final int maxSpeed;
  private final int minSpeed;
  public Jet(String callsign, int maxSpeed, int minSpeed){
    this.callsign = callsign;
    this.maxSpeed = maxSpeed;
    this.minSpeed = minSpeed;
  }
}
```

Now its clear what these things mean and why they belong together. Any place a
`Jet` is used I will immediately know what the data I am interacting with
represents and will probably know why I am doing it.

Eventually I realized that I was spending more time in dynamic languages trying
to figure out a few cryptic, tersely lines of code than I was actually writing
new code. And even though it takes me perhaps 2 times longer to write code, I
don't actually spend a lot of time writing code. I spend a lot of time
communicating, planning, debugging, testing, and only then finally actually
writing code.  Optimizing such a tiny part of the application creation process
was not only unimportant, it was actually hurting my ability to do the other
things that I do, which take 70% of my time vs. the 30% spent writing actual
code.

### Action, State, and Communication

I think of modules of well-designed modules of DI code can be divided into
three categories.  Each category of object is represented in Java code by a
class implementation. These categories are strictly ways of thinking about what
an object is and what it should do - there is no formal implementation of them
in any library or any part of the Java class path (although some languages do
attempt to implement similar concepts formally within the language).

- Action objects (or as I often call them, do-er objects, or functions) are
  objects that do something. They often have only one method in them. Action
  objects often rely on other action objects in order to perform whatever they
  do. This is called a dependency, and is usually expressed in Java by making the
  dependency a constructor argument.
- State objects keep track of application state, and usually rely on a large
  number of action dependencies effectively encapsulate and represent the
  manipulations that they apply over some sort of memory (ie. hard disk, RAM
  contents, or network resources).
- Message objects have no behavior, only state. They do not have any
  dependencies on any other objects at all. I also sometimes call these objects
  _named tuples_. This design pattern in Java is also often referred to as a
  POJO, which means plain old Java object.

#### Communication/Message Objects

##### Finality

Making the fields of any objects `final` is a good way to remind yourself what
the message is and what the requirements are for instantiating it. By making
fields final, you ensure that the object doesn't change after it is
instantiated, and this property is strongly enforced by the JVM.  Finality
avoids many common problems that are very difficult to debug.

##### Fluent Building

You may often find that the creation of such objects is burdensome, especially
if you want to establish special rules around their creation such as optional
arguments or defaults. A traditional way to accomplish this is through
constructor polymorphism; however this can be difficult. Take, for example, an
object that has various defaults you might want to express for 10 fields. In
order to have a unique constructor for every combination of fields that might
be set or not set, you need 2^10 constructors. Thats crazy!  A common design
pattern in Java for creating objects of this sort, as well as stateful objects,
is a _fluent builder_.

A fluent builder is a special factory, where each method sets a value to be
passed to a constructor.  This allows users of an object to efficiently specify
arguments, while also managing default value logic and error checking in a more
compact manner. An example of a builder is below:

```java
public enum EngineType { RAM, SCRAM, TURBO }

public class Jet {
  ...
  private Jet(EngineType engineType, String callsign) {
    this.callsign = callsign;
    this.engineType = engineType;
  }
  public static Builder newJet(){
    return new Builder();
  }
  
  public static class Builder {
    private String callsign = null;
    private EngineType engineType = EngineType.TURBO;

    public Builder withEngine(EngineType engineType){
      this.engineType = engineType;
      return this;
    }

    public Builder withCallsign(String callsign){
      this.callsign = callsign;
      return this;
    }

    public Jet build() throws JetCreationException{
      if(this.callsign == null){ 
        throw new JetCreationException("Jet callsign cannot be null");
      }
      return new Jet(engineType, callsign);
    }

  }
}
```

An invocation of the above example might look like this:

```java
Jet jet1 = Jet.newJet().withEngine(RAM).build(); // throws exception
Jet jet2 = Jet.newJet().withCallsign("JK-2").withEngine(RAM).build(); // ok
```

As you can see, we have created a situation where we have a lot of nice
polymorphic properties that just aren't part of an official language spec. If
we ever need to change the logic of how jets are created, there's no static
references to constructors to trace back and update. Also note the use of
[precise exception handling](./exceptions.md)).


#### Example of an Action/Function/Do-er

Notice that this class only exists to host a method, making it effectively a
wrapper for a function.  Despite the fact that classes may have many methods,
not all classes _should_ have many methods. The easiest machines to maintain
are the ones with the fewest screws :).

```java
class CharArrayLowercaser {
  public char[] lowercase(char[] toLowercase){
    char[] newChars = new char[toLowercase.length]);
    for(int i = 0; i<toLowercase.length; i++){
       newChars[i] = Character.lowerCase(toLowercase[i]);
    }
    return newChars;
  }
}

```

In the class below, we do have some fields, but they are other actors,
therefore the code is actually stateless. This is a valuable property not just
for serial application design, but is also extremely in distributed systems.
The user interfaces of most distributed applications that I implement are
designed to be stateless.

```java
class JetMaker {

  private final CharArrayLowercaser lowercaser;
  
  @Inject
  public JetMaker(CharArrayLowercaser lowercaser){
    this.lowercaser=lowercaser;
  }

  public Jet makeJet(char[] name){
    return new Jet(lowercaser.lowercase(name));
  }
}
```

A few more notes about the above implementation:

1. The JetMaker is a special type of action that creates stateful objects or
 messages. This is a common design pattern called a factory.
2. The validation of the message object (basically, making sure the name is
 always lowercase) is carried out in the action, not in the message constructor.
 There are benefits and disadvantages associated with this approach: when
 working with dirty data, you will often want to be able to instantiate objects
 which may be invalid in order to inspect them and provide user feedback. On the
 other hand, you invite error in your application by allowing state to be
 created without validation, and must now rely on everyone being aware of the
 appropriate validation libraries in order to ensure correct application state.
 How can you use Java's code visibility features to reduce the impact of such
 decisions?
3. The parameter for a factory method is sometimes called a runtime dependency - 
 this terminology is meant to draw contrast to the common DI practice of
 injecting dependencies once - making an application's configuration effectively
 static, even if it doesn't rely on Java's `static` keyword to do so.

#### Example of a State Object

```java
class JetStore {

  private ObjectMapper mapper = new ObjectMapper();
  private final File jetFile;

  @Inject
  public JetStore(@Named("jetstore-location")File jetFile){
    this.file = jetFile;
  }

  public synchronized void storeJet(Jet jet) {
    List<Jet> currentJets = mapper.readValue(jetFile, new TypeReference<List<Jet>>(){});
    currentJets.add(jet);
    mapper.writeValue(jetFile, jet);
  }

}

```

The object above uses a file to store state. This makes it stateful.  The
actual file that the object uses to store state is an injected dependency.  The
`@Named` annotation here is very important: it allows Guice to distinguish
between all the different objects that might be bound to this particular site.

When we actually bind the file that we would like to use, we will likely use an
object of type `String` rather than an object of type `File`. An illustration
of this binding might look like this:

```java
  bindConstant().annotatedWith(Names.named("jetstore-location")).toInstance("file:///...");
```

When Guice searches for bindings to the object of type `File`, and sees that
there is a constant bound with a matching name, it will look for a
`TypeConverter` to convert from a `String` to a `File`.

All type converters actually convert from String to some other type, and are
specifically for binding constants (which in Guice are always `String` objects)
to other, richer type instances.

To register a `TypeConverter` implementation with Guice, you need to use the
`convertToTypes(Matcher<? super TypeLiteral<?>> typeMatcher, TypeConverter
converter)` method in a Guice module to bind the `TypeConverter` to the various
types it is able to convert.  Since Java features a rich type model, you might
not want a type converter to convert all of the subclasses of a class, or you
may need to perform runtime evaluation of generics to ensure that conversion is
actually possible. As a result, the registration also requires a `Matcher`
class which tells Guice what types the converter is able to convert to.

Patterns for Leveraging Java Serialization in Distributed Applications
----------------------------------------------------------------------

Serialization is an important topic in distributed applications because it
provides the mechanism by which we deploy application state to multiple
physical hosts and JVMs.

While there are many good serialization frameworks out there, Java
comes with one that is built-in, and which is fairly light-weight
in terms of code (although, it is often maligned for its processer
inefficiency).

Because native Java serialization is lightweight to work with in code,
and because application start-up times are rarely a significant issue
in distributed applications, Java serialization is great for use in application
start-up type use cases.

Storm, for instance, allows you to use Java serialization with Bolt and
Spout objects (which are important programming primitives within Storm).
Rather than using a map of configuration values to populate the state of
a Storm Spout or Bolt, you can simply use Java serialization. This is
exceptionally convenient for use with Guice, because you can use Guice
injection to configure a Storm topology rather than writing boilerplate
configuration code.

One encounters difficulties in this paradigm, though, when one begins
using stateful objects that cannot be serialized, such as database
connections.

This problem can be solved by using the factory design pattern, making
sure that the actual factories themselves are serializable.

Take the example below:

```java
public class JetRadar {
  private DatabaseConnection connection;
  
  public void logPosition(Jet jet){
    connection.insert(jet.getLatitude(), jet.getLongitude());
  }
}
```

Here, it is unlikely that we can serialize `DatabaseConnection`. Rather than
doing that, though, we can tell the application how to create a new 
`DatabaseConnection` any time it needs (such as when it starts up on a new
remote host!).

This example works better in such a context:

```java
public class DatabaseConnectionFactory implements Serializable{
  private final String username;
  private final String password;
  private final String uri;
  
  @Inject public DatabaseConnectionFactory(
      @Named("db-username" String username, 
      @Named("db-password") String password, 
      @Named("db-uri") String uri){
   ...
  }
  
  public DatabaseConnection getConnection(){
    return new DatabaseConnection(username, password, uri);
  }
}

public class JetRadar implements Serializable{
  private final DatabaseConnectionFactory connectionFactory;
  private transient DatabaseConnection connection;
  
  @Inject public JetRadar(DatabaseConnectionFactory factory){
    ...
  }
  
  public void logPosition(Jet jet){
    if(connection!=null){
      connection = connectionFactory.getConnection();
    }
    connection.insert(jet.getLatitude(), jet.getLongitude());
  }
}
```

Now, when we serialize `JetRadar` and transfer it to a remote host,
it will be able to create a new connection using the connection
factory.

It's important here to take a few moments to talk about security
and state handling. 

Since database connections (and their credentials)
are one of the main targets for this sort of thing, care must be
taken to make sure that you do not compromise your credentials by
serializing them to a location that is readable by others. Databases
that leverage Hadoop's delegation tokens are favorable in this
context because the system has been designed for exactly this type
of use case. Kerberos tokens share a similar useful property, although
they can be more difficult to leverage because of the host-based
security checks in the Kerberos protocol.

You also must use good connection handling practices when using
these patterns. Often an application will start and leave a connection
open for many hours, however there are not always convenient ways
to close a connection as part of the application lifecycle. Storm,
for instance, provides no shutdown hooks to signal Bolt shutdown.
In such situations, you can leverage the JVM's own lifecycle guarantees
via the `java.lang.Runtime.addShutdownHook()` facility.

