Data Models
===========

This is intended to be a TL;DR for folks who are interested in learning
about data modeling, specifically in the "big data" context that has
become so popular, and which is often positioned as being somehow fundamentally
different from the transactional/relational database architectures that have been
popular since the 70's.

There are many great resources on data modeling available. My favorite is
Chris Date's book _Database In Depth_, published by O'Reilly. It is a
fairly lightweight and practical treatment of relational theory, absent
the clutter of particular vendor implementations of the theory. This is
the most helpful resource to folks working in unstructured databases,
because (as I will discuss below), these folks will wind up implementing
much of this theory on their own.

I strongly suggest that anyone working with an unstructured database
such as HBase, Mongo, etc. become well-aquainted with the existing field
of information management. Especially relational theory and data marting
techniques such as star schemata. Learn about the role of ETL in traditional
enterprise architecures. I will offer some background on these topics below
but there is much more on the internet in terms of conference and user
group decks on Slideshare, etc. that will serve to further illustrate and
convince you of the ideas that I outline below.

There's No Such Thing as "Unstructured Data"
--------------------------------------------

Lets start by getting clear on terminology.
* _Relation_ is a term derived from the field of mathematics. It means,
  rougly, a row in a table. It does not mean a foreign key relationship or constraint 
  between tables, as many folks often assert.
* A _transactional_ database is a database that is meant to accept one or
  more modifications to the database as an atomic event that is consistent
  between all users and connections to the database.

Using the above definitions, we can assert that the primary difference
between a database such as Mongo or HBase is not that the database is not
relational - in fact the relations that these databases store are of
type (string) and type (bytes, bytes, bytes, long, bytes) respectively.

The main difference is in the fact that the databases work over a fixed
type, that they do not support transactions (or have limited or reduced
transaction support), and that they have limited or reduced relational
query support.

Many "NoSQL" databases _do_ have enhanced support for certain types of queries
that are not defined using relational algebra, such as search over documents.
These features may be of significant value to an application architect because
they allow that architect to optimize certain application-database interactions.
Others may find that the increased scalability features offered by such databases
far outweigh the convenience offered by having built in joining, grouping, aggregate
functions, etc.

Many relational technologies also exist that provide query support over
a variety of data sources, and may be bolted on externally for reporting purposes outside
of the application-data API interactions.

### About Reduced Transactional Consistency Support

Assume the following:
* I begin scanning (reading in bulk) a table at 03:33:12 UTC and that read operation runs 1 hour
* A user deletes a record at 03:34:12 (one minute after my read operation begins

In a fully ACID or "Transactional" database my scan operation is **I**ndependent and **C**onsistent
with the time that the scan started. I see the world exactly as it was when I requested the data.
This means that the record the user inserted will be reflected in my scan, even though they
may have deleted it by the time the record actually goes over the wire and is read by my process.
It is important to note that independence is _frequently_ relaxed in _all_ database implementations.
There are very few databases that can truly be said to be ACID. This is simply because
fully independent queries are difficult to implement and may have severe implications on
database performance.

In a not-fully ACID database (HBase, Cassandra and a number of SQL databases come to mind) I
will not see the record, as the database does not guarantee that it will keep track of these sorts
of things.

It is important to note that many database implementations that claim to be fully ACID will
actually not do the Right Thing in this situation either. Caveat emptor.


### About Reduced Query Support

Relational algebra defines three fundamental operations:
* Projection (equivalent to the SELECT clause of a SQL statement) means
  to reduce the set of elements in a relation
* Selection (equivalent to the WHERE clause of a SQL statement) means
  to filter or reduce the number of records from a table
* Substitution (sort of similar to a JOIN clause in a SQL statement) means
  to use values from one relation to substitute values in another relation.
  _Aside: substitution, projection, and selection all work together to define
  the various flavors of join operations such as natural, left, right, outer, inner,
  etc.)

Given these definitions, the databases above actually have very robust support for
projection and selection operations. The creators of these databases have simply
omitted support for the substitution operation. Austensibly, it is the role of
an application or framework developer to implement these, should they be needed.

Common Data Analysis Architectures + Models
===========================================

It's hard for me to separate data analysis architectures and models because the most
common architectures are focused around using a particular modeling approach at a
particular layer of the architecture. So I'll discuss them at once.

Below I will use the term _transactional_ to mean a database that is primarily intended
to serve an interactive application. This is subtly different than the definition I
offered above, because not all interactive applications need to be transactional 
(although they usually are). All interactive application databases _do_ however need
to be isolated from bulk workloads. This is the main purpose of all data analytics
architectures.

A Word About Ongoing Operations
-------------------------------

The world around us constantly changes. As a result data changes. New data comes
and old data goes away. The assumptions of the real world that you build a data model on
will probably no longer be valid within a year. You should be prepared to make data
modeling, managing, and analytics as routine as the janitor's evening rounds.

Data modeling is an ongoing activity. Many organizations have focused on episodic
requirements management models (eg. waterfall) in order to manage their data warehouses. 
It is my personal belief that in order for an organization to take maximum advantage of
it's data, it must be prepared to engage in ongoing modeling and management activities.
It must be prepared to respond quickly to changes in what information is needed and
what information is available.

Using agile requirements management methods such as Scrum or Kanban are ideal for this.
Many volumes have been written on agile methodology, so I won't belabor the finer points
that they make. Some key take-aways (in my mind) are:
* Start with a plain-vanilla bare-bones methodology. If you use JIRA, for instance, and
  you start customizing issue types, etc. right out of the gate, then you're doing it wrong.
  If you have previously used another methodology then Agile will **not** come naturally
  for you and you will need to force yourself to adopt it. If you do not feel as though
  you're challenging yourself you're doing it wrong.
* Customize, adapt, and change only when you identify problems. All the agile methodologies
  I'm aware of incorporate regular retrospective events where you should do this. Do
  not change too many things at once.
* Get a real Agile coach. This is someone who has successfully employed Agile practices
  somewhere and will be able to provide you positive constructive feedback.
* Agile does not mean unstructured. Agile methodologies are usually very structured. In
  my experience, however, their structure often closely matches that of reality 
  (which is not often the case for other methods). If you think that agile means there
  are no scheduled meetings, no requirements documents, or no formal design review
  or change control meetings, then you're doing it wrong. Agile means you adopt a 
  usually highly structured process that can rapidly change to meet reality.

**The** ELT Architecture
------------------------

This is rapidly becoming the most prevalent architecture for data analysis management
and reporting. The reason for this is because it is repeatable, it bears similarity to
other previous architectures and is quickly relatable to veteran data architects, and
it is very flexible and can be used to rapidly solve many real world problems.

### History

The term ELT is intended to draw contrast with ETL, another common architecture. In ETL,
data is loaded extracted from a transactional source (SQL or other), transformed
(using the computational power afforded by the source database, and then loaded into a
data warehouse.

The problem with this approach is that the transformation step may begin to impact the
performance of the source database, if a large number of transformations are required
or the transformations that are required begin to take a long period of time as the
database grows.

### Application

In an ELT architecture, data is extracted (ie. read in bulk) from a transactional database
and immediately loaded, untransformed, into a data warehousing environment. _This is
critical_ because this one change provides essentially all the benefits afforded by the architecture.
By reading data in bulk, and _not tranforming data using the transactional database's query engine_, we
reduce the load on the source database. 

This has the following benefits:
* You may create a proliferation of views over the original data. This makes creating the "right"
  model the first time around (and understanding the details of the various normal forms such
  as 3nf, etc.) unnecessary. All you need to do is have a team of data engineers who can create
  views that satisfy the needs of various user communities.
* All the data for an entire organization can be placed in one data warehouse (often popularly
  referred to as a "data lake"). It is much easier to: find data; create reports that
  relate the activities of different departments, sub-organizaions, etc.; and manage the security
  policies for analysts or "data scientists" across a single homogenous reporting platform.
* Load on a transactional database is *dramatically* reduced, so it can better go about the business
  of serving customers, users, etc.

Implementing an ELT architecture is not just a technology problem. It is a requirements management
and organizational problem.

The common layers of this architecture are:
* Ingest
* Modeling
* Indexing + analysis
* Reporting
Each layer is discussed in turn below.

#### Ingest

Ingest (or extract) is simply reading data from a source (transactional or other) and storing
it in the data warehouse's catalog of raw data. There are many tools to do this today. Please
see the appendix on ingest tools for more info on technologies that may be used to do this.

Ensuring ingest consistency is a common problem in these architectures, because in the real
world ingest processes fail intermittently. A failed ingest may actually have catastrophic,
rippling impacts on the various modeling, indexing, and reporting layers because for example, if
a failure impacts a daily report, and the impact is not noted, a person may make an incorrect
or ill-informed decision. Imagine if, for instance, an airline fails to receive accurate weather
information, or uses information that is 24 hours old. They could waste millions of
dollars in one day of cancelled or diverted flights.

There are tools now to help you manage and report on the SLA's for different data feeds (see
the below appendix on ingest tools).

Most of these tools require that ingest operations be consistent and idempotent (they always
produce the same result and you can run them twice and no harm is done). One common way to
implement this is using staging directories. An example layout for data ingest might be, for
instance:

* `/data/incoming/<feed_name>/<feed_date>`
    - A temporary location where incoming data is written by an ingest process. If the ingest
      fails before it is complete, then this directory may be deleted.
* `/data/archive/<feed_name>/<feed_date>`
    - A final directory, where incoming data is written when a successful ingest event occurs

Many data warehousing technologies now support partitioning and compression techniques to make
retrieval over historic data faster, by placing data that is (for instance) over a year old
in a different location than data from the last year. That way, a relational engine may inspect
the partitions in the data and not even bother reading in data from last year if only data
from this year is desired.

Ingested data should almost always bear the same schema in the data warehouse that it did
in the original (often transactional) source.

#### Modeling

Raw data is usually messy. It can be difficult to draw meaningful conclusions from because:
it does not share the same units, identifiers, etc.; it contains duplicate records or irrelevant
information; or simply because it is poorly organized.

Modeling is basically the process of creating views of raw data that make sense. The process
of modeling in an ELT environment is much the same as it is in any other traditional data
warehousing environment. You want to focus on creating entities and relationships. You may
document these via ER-diagrams. You should have some knowledge of the various normal forms
that have been proposed in relational theory and try to adhere to them as appropriate.

Unlike traditional data warehousing, you do not need to focus too hard on getting the model
right for all possible use cases, however. Just the ones presented by significant stakeholders.
In a large company or government agency, for instance, this may mean that there are a small
number (on the order of 1's-10's) of independent modeling efforts establishing entities
and their relationships.

#### Indexing + Analysis

Two common use cases for data are indexing (search, knowledge management + discovery, etc.) 
and analysis (summarization of facts, forecasting, etc.). This is where the most value is
derived, and is typically where organizations incur the highest costs.

Search and data marts are the main ways data is exposed in this layer.
* Search is creating a search engine over data. This is like Google.
   - Search is complicated. Many organizaitons have specific informatics tasks that they
     may need accomplished in a search engine. Do not take for granted that search is
     simply a Solr server and a few batch jobs to upload data to the Solr server.
* Data marting is the process of creating online analytic processing models of your entity
  relationships (often called OLAP cubes). These models are provided so that users may
  rapidly summarize different faceted facts about specific entities that exist in your larger
  entity model.
   - As an example, someone may wish to report information about sales figures. They may wish
     to summarize information by state, county, different periods of time, types of products,
     who purchased them, etc. In order to do so efficiently, it is good to provide them
     a specific model that allows them to rapidly find the facts that they need rather than
     asking them to understand your complete entity-relationship model (which may be very
     complex).
   - This is often accomplished using something called a star or snowflake schema. There
     Are many database products that specifically optimize the types of computations
     that are needed when querying such schemata. See the relational query engines
     appendix below.

#### Reporting

The reporting layer of the ELT architecture is for providing information summaries to
non-technical users. This often takes the form of a search or dashboard application, but
may be as simple as a Powerpoint presentation delivered to an stakeholder.

Reporting is now often accomplished with a combination of e-mail, REST services, and modern HTML
5 applications.


Appendix
--------

### Relational Query Engines

There is an increasing separation between storage technologies and relational query
technologies. Most engines that have been developed in this area target analyst-data
interaction (meaning it is intended as a component for analysts to interact with
when generating reports, etc.), however one in particular (Phoenix) focuses on application-
data interaction (meaning it is intended as an API for application developers to
interact with using Object-Relational-Mapping tools or API's like JDBC).

* Pig is a relational query technology that can be layered over large flat files
  or sources such as HBase, Mongo, etc. Pig implements a SQL alternative called
  Pig Latin, which is arguably closer to the original Relational Theory than
  SQL. Many people find Pig Latin hard to learn, though.
* Hive, Drill, and Impala are all relational query engines that use a SQL-like interface
  and conform to various versions of the HiveQL language. Drill and Impala
  implement special optimizations for data that is frequently grouped and aggregated
  across a low-cardinality sub-set of dimensions. This makes them ideal technologies 
  for data marting (more on data marting later).
* Phoenix is a relational indexing and query technology built on HBase that uses a SQL
  variant. Unlike the other technologies mentioned it is tightly coupled with HBase and
  may not be used without HBase. It is primarily intended to support application-data 
  interaction rather than analyst-data interaction. Interaction with Phoenix is very 
  similar to interaction with other transactional databases, and may be more familiar 
  and easy to understand than direct API access to HBase for many developers.

These engines allow you to plug a robust relational query engine on top of almost any
data source you can envision, and also help separate application-data interaction
from analyst-data interaction. This feature may be desirable for many application architects
who find benefits in the native database API and query support, but still need ways
to perform bulk reporting and extract-load-transform (ELT) over data.


