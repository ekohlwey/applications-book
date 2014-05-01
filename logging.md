Logging, Metrics, and Diagnostics
---------------------------------

Logging and metrics gathering are important for _diagnosing_ problems with a
production application. I say diagnosing, because you really should never be in
a _modify, build, deploy, run_ debug loop in production - **ever**. If you are
you are doing something wrong.

Logging
=======

Within the Java ecosystem, there is a suite of de-facto standards for logging.
These are, roughly, [SLF4J](http://www.slf4j.org/), 
Log4J[1](http://logging.apache.org/log4j/1.2/) and
[2](http://logging.apache.org/log4j/2.x/), [LogBack](http://logback.qos.ch/),
and [Commons Logging](http://commons.apache.org/proper/commons-logging/).

### Choosing a Logging Interface

The first step to getting logs is to choose a logging interface. You may choose
among any of the projects listed above for a logging interface, however the
most commonly used interfaces are SLF4J and Commons Logging.

Almost all of these frameworks have an interoperability layer with each other, so
in the end what front-end you use has little impact on the back end you use - 
you may simply find the interoperability more difficult to configure. Of all these
projects, SLF4J is the only one that has been explicitly designed as a
front-end interface, so it is probably the easiest to work with from
an interoperability standpoint.

### Choosing a Back-End

This is often not a choice. Many frameworks, such as Hadoop and Storm, come
packaged with a particular logging back-end. You may elect to switch it out, but
you may find this task difficult.

Most logging back-ends are able to interoperate with multiple _appenders_. An
appender is just an implementation of how to actually log events, so you
will see console appenders, file system appenders, etc.

When operating a distribtued application, it can be difficult to inspect logs
because they are scattered acrossed many physical servers. A simple way to fix
this is through log aggregation.

#### Aggregation

The main tool for doing this at the moment is
[Flume](http://flume.apache.org/FlumeUserGuide.html), although there have been
many proposals to transition to Storm for log aggregation as well.

In most deployments, Flume will be configured to watch a directory with
uniquely named files. As files are created and appended within this directory,
Flume will make a best-effort to transmit and aggregate their contents to a
central logging facility (Flume calls this a sink).

It is important to keep in mind that the durability of Flume sources and
sinks are subject to the durability guarantees of the implementation. Even
though Flume uses an agent which is very good at keeping track of transactional
aggregation state, the framework's durability guarantees are subject to the
devices it is implemented over. Nothing is 100% durable.

You may configure Log4J 1.2 to create reasonably unique names by using the 
`log4j1.2-extras.jar` and configuring the appender to use time and daemon 
based file names. For instance:
```xml
<rollingPolicy class="org.apache.log4j.rolling.TimeBasedRollingPolicy">
  <param 
      name="FileNamePattern" 
      value="/var/log/<appname>/<daemonname>.%d{yyyy-MM-dd_HH-mm}.log"/>
</rollingPolicy>
```

Flume may also be used to aggregate Syslog events that are transmitted with TCP
or UDP appenders.

Recent releases of Flume include appenders that work natively with Log4J.

#### Impact of Logging

Logging can have a dramatic, negative impact on IO resources, at whatever level it
is implemented (usually disk or network, but sometimes also even processor and
system bus). All logging frameworks contain logging levels in order to help organize
logging events by a general importance and impact category. These are usually
_trace_, _debug_, _info_, _warn_, and _fatal_. Most logging frameworks also include
the ability to define a custom level that you can use for purposes such as audit
logging.

When logging, you should always guard logging statements to see if the level is
enabled. Doing so ensures that computation, such as string concatenation, are
not performed. While this may not seem like a big deal, I have personally seen
2-3x performance reduction effects from poorly implemented logging statements.
Here's an example of how to log correctly:

```java
class JetFactory {
  static final Logger LOG = LoggerFactory.getLogger(JetFactory.class);
  ...
  public void checkLaunchpad(int padNumber){
    if(LOG.isDebugEnabled()){
      LOG.debug("Pad number " + padNumber + " checked.");
    }
...
```

You should also take into consideration _what not to log_.

### What Not to Log

I say what not to log, because almost anything may be of interest at any given
time. So the question to ask is not so much what to log, but what things should
you never log, or what are common bad practices for logging?

Every time you log something, you transmit anywhere from 10-100 bytes to an IO-
constrained resource. Maybe that resource is a physical disk, or maybe it is a
network adapter. Regardless, the important fact to keep in mind is that this
resource is _constrained_ and your logs must always be a secondary consideration
to the performance of the application being logged.

You should always be cautious of using logging to diagnose performance issues, 
for instance, because logging itself creates performance issues. If you do
use logging to diagnose performance issues, you should make it as specific
as possible using a combination of logging levels and class or package based
level configuration.

#### Level and Package Filtering Configuration

All logging frameworks have methods to detect if a certain log level is enabled,
and for which part of the code base that level is enabled.
You can easily recognize such methods in _guarding statements_, which usually
look something like this:

```java
class SomeClass {
...
// this is a log declaration
private static final Log LOG = LogManager.getLog(SomeClass.class);
...
    // this is a gurard statement
    if (LOG.isDebugEnabled()){
      LOG.debug("Something happended at " + System.currentTimeMillis());
    }
...
```

By convention, the `LOG` object above is scoped to a particular part of the codebase
or aspect of logging, usually by the fully-qualified class name. This is usually
achieved by using a single static instance that is declared within a particular class.
The logging framework is able to detect where a logging statement occurs and at 
what level it is logged though the combination of the class name used in the
static field declaration and the method signature of the guard statement.

The logging framework is coded to be very efficient in determining whether or not to log.
By doing this, you are able to avoid the overhead associated with a particular
logging statement.

You can achieve different levels of scoping on a per-class or per-package basis
in Log4J 1.2, for instance, using a configuration like this:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE log4j:configuration SYSTEM "log4j.dtd">

<log4j:configuration xmlns:log4j="http://jakarta.apache.org/log4j/">
  <appender name="console" class="org.apache.log4j.ConsoleAppender"> 
    <param name="Target" value="System.out"/> 
    <layout class="org.apache.log4j.PatternLayout"> 
      <param name="ConversionPattern" value="%-5p %c{1} - %m%n"/> 
    </layout> 
  </appender> 

  <root> 
    <priority value ="warn" /> 
    <appender-ref ref="console" /> 
  </root>
  
  <logger name="com.mycompany.apackage.SomeClass">
    <level value="debug"/> 
  </logger>
  
</log4j:configuration>
```

The first XML element, `appender`, declares a new appender called `console`, and
sets up the actual implementation of how the console appender will append. Each
appender will have its own implementation-specific configuration options.

The second XML element, `root` sets up the root logger. In Log4J, all loggers
inherit from some other logger - if one is not specified then it is assumed to be
the root logger. The only configuration here is the priority to log and the
appender to attach to.

The last XML element, `logger` sets up a logger for our `SomeClass` class, and allows
us to see debug statements _only from that class_. Since all other loggers are
configured (via the root logger) to log at `LogLevel.WARN`, the only class we will
see `LogLevel.DEBUG` information from is `SomeClass`.

### Where to Log

Deciding where to log has important implications on application performance and
the durability guarantees that you may expect from your logs. Consider the
following:

* Log events are usually queued in memory and only flushed periodically by a
  logging thread in order to minimize the impact of logging on application
  performance.
* Logs persisted in memory are only as durable as memory and the process that
  hosts them
* Local disk is usually where you put logs because it has relatively high durability
  and relatively high IO available
* Logs persisted to local disk are only as durable as the local disk
* Rotating logs may be rotated off of disk before an aggregator can transmit
  all logs to a central logging resource, if logs are logged in high volume
* TCP can be used to provide better logging durability guarantees at the cost
  of network bandwidth
* UDP can be used to provide intermediate durability guarantees between those
  of disk and TCP, with proportionately lower impacts on network IO.

Given all of the above considerations, you may find that different types of logs
need to be written to different places. Luckily, logging frameworks in general
have you covered: it is generally easy to configure one or more types of
appenders for a given application. Doing so, however, will dramatically complicate
the implementation of aggregation, so use this capability with caution!

Metrics: an Alternative to Logging
==================================

Metrics gathering is, arguably, a special case of logging. Metrics gathering
is typically a lighter-weight process than standard logging, and heavily 
dependent on the time dimension.

The de-facto metrics gathering system today for distributed systems is
[Ganglia](http://ganglia.sourceforge.net/). As a Java user, you will probably
find the [Coda Hale Metrics](http://metrics.codahale.com/) API for Ganglia
convenient to use with tools such as Ganglia. 

Metrics are usually a simple tuple of `(time, number)`. Many metrics frameworks
will aggregate this information to provide basic statistical properties of 
a particular sample window, such as mean, median, and quantiles.

Many frameworks contain a metrics API that you should consider using if you are
writing a plug-in or component for that application. Hadoop Map/Reduce, for
instance, has a counter facility that you can use to report metrics about
your executing Map/Reduce job.

### When to Use Metrics

Metrics are convenient any time you need to report numbers - they are lighter
weight and faster than logging.

Metrics may also be the _right approach_ to begin with. Many aspects of application
state, such as heap size, object allocation, etc. can be very difficult to report
in a meaningful way using text. Without logging a full heap dump, or
printing the entire contents of a data structure, it is very difficult to report
on many important aspects of application state without using a number.

Whats more, by using a metrics framework, which correlates all numerically-measured
metrics of your application to points in time, you dramatically enhance your
ability to diagnose and understand application state.

Take, for example, a search application that is running slow. A log-based
approach for this might log query execution time, or try to insert logging
statements in a tight loop within the application. This is a very constrained view
of the system architecture, because we are unable to understand the performance
of system resources or other daemons that may be executing within the cluster
(on the same physical host or other hosts).

A metrics based approach to performance diagnostics, however, allows you to easily
compare query execution time to disk IO, network IO, processor load, and RAM
utilization (among many other metrics!). This ability to correlate time series
and test hypotheses "in-vivo" is unmatched by logging.

Diagnostics
===========

When diagnosing problems with a system at scale, be they performance,
stability, or data related, you should leverage logging and performance only in
as much as it is necessary to understand the problem. If you enable detailed
logging such as debug logging, you should turn it off once the problem is
identified. You should be very cautious not to introduce logging or metrics 
gathering in tight loops where it is likely to create its own performance problems. 

Once a problem has been diagnosed, you should be able to implement a test on
your local machine that replicates most problems. If you can not easily write a
test that diagnoses a problem, then you should consider refactoring your code.

An example of such a situation might be a race condition that arises when two
pieces of code are run at the same time:

* If part of the code is threadsafe, but of of it isn't:
    - Consider refactoring the code into more "doer" classes using dependency
      injection techniques.
* If the code is not threadsafe:
    - Make it threadsafe by adding locking constructs. This may itself often
      require a significant rework of the code.

Once you identify a problem, you should use [testing best practices](./testing.md)
to ensure that it does not happen again.

### Diagnostic Based Alerts

If you discover a condition that results in a potential problem for your
application or cluster, you should write an alarm to check for that condition.
There are many alerting frameworks - one of the most popular today is
[Nagios](http://www.nagios.org/) and it's fork
[Icinga](https://www.icinga.org/). Amazon, in particular, also offers a suite
of services called [Cloudwatch](http://aws.amazon.com/cloudwatch/).

This practice can be compared to regression testing, where you write an automated
test to let you know that a regression has not occurred.
