Design Patterns
===============

Below, I document some object-oriented design patterns that I have developed or observed, that
I believe to be useful but am not aware of adequate documentation elsewhere. If you are
aware of references, please let me know!

Plain Old Java Aspects
----------------------

Aspect-oriented programming is about identifying cross-cutting concerns in your
application, and creating code reuse around thsoe cross-cutting concerns (called
aspects). There is a helpful library that you can use to do this, called AspectJ,
however you can also do something very similar in plain old Java using decorator
patterns and thread local variables.

Aspects are defined using a descriptor called a pointcut, which is basically
a description of a particular pattern of method invocations or stack state.
AspectJ uses a rich language to express pointcuts, but you can also do this
manually in Java simply by writing the code that you want to execute at the
beginning or end of a method.

For example, if I want to run the method `ensureJetState()` every time a method on
a `JetFactory` is invoked, I can simply write it inline in each method.
Notice that, below, the aspect we are creating relates to the cross-cutting concern
of creating jet state. This concern involves not only the setup, but access of state.

```java
class SupersonicJetFactory implements JetFactory {

  JetState state = null;

  void ensureJetState(){
    if (state == null){
      state = new JetState();
      // other setup
      ...
    }
  }

  void liftPart(){
    ensureJetState();
    Part part = partWarehouse.fetch();
    crane.liftPart(part);
    crane.attachPart(part, state);
    ...
  }

  void dropPart(){
    ensureJetState();
    ...
  }

}
```

This can get tiresome to write for all the different implementations of `JetFactory`.
Instead, I can use a decorator design pattern to make a reusable component that
encapsulates one half of the aspect.


```java
class StatefulJetFactory implements JetFactory {

  JetState state = null;

  void ensureJetState(){
    if (state == null){
      state = new JetState();
      // other setup
      ...
    }
  }

  void liftPart(){
    ensureJetState();
    delegate.liftPart();
    ...
  }

  void dropPart(){
    ensureJetState();
    delegate.dropPart();
    ...
  }

}
```

Notice that this actually only encapsulated one half of the aspect, though - the
setup part. The state still needs to be accessed.

One way to overcome this issue is to make the state a method parameter, for instance
by modifying the `liftPart()` method to become `liftPart(JetState state)`. This
is sloppy because it introduces new method parameters and makes the code brittle
as new aspects are introduced.

Instead of introducing a new parameter, I can also use the state of the call stack
to describe the state of the aspect (as AspectJ does). This can be done using a
`ThreadLocal` variable and static method.

```java
class StatefulJetFactory implements JetFactory {

  ThreadLocal<JetState> state = new ThreadLocal<JetState>();

  void ensureJetState(){
    if (state.get() == null){
      state.set(new JetState());
      // other setup
      ...
    }
  }

  void liftPart(){
    ensureJetState();
    ...
  }

  void dropPart(){
    ensureJetState();
    ...
  }

  static JetState getState(){
    JetState state = state.get();
    if(state == null) {
      throw new IllegalStateException("Jet state can only "+
          "be retrieved within a properly-structured aspect "+
          "invocation.");
    }
  }

}
```

Then, to use the `JetState`, completing the second half of the
aspect:

```java
class SupersonicJetFactory implements JetFactory {

  void liftPart(){
    JetState state = StatefulJetFactory.get();
    Part part = partWarehouse.fetch();
    crane.liftPart(part);
    crane.attachPart(part, state);
    ...
  }

  void dropPart(){
    ensureJetState();
    ...
  }

}
```

### Other Examples

Other examples of this pattern are present within the JDK, specifically
in the implementation of JAAS, using the `Subject.doAs(...)` method to
pass around credential information as part of a cross-cutting concern.

