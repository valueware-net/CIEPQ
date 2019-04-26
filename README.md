# CIEPQ Engine

TL;DR

A CIEPQ Engine is a platform for building software:



*   With amazingly high performances through concurrency and distribution
*   Thanks to horizontal and vertical scalability
*   Naturally resilient and fault-tolerant
*   Rapidly changeable and evolvable
*   For complex business requirements.  

It gets its name by CIEPQ, a method for modeling business processes in software.

It comes from an evolution of CQRS and Event Sourcing together with Domain Driven Design in a Hexagonal architecture. It shares some important features with Stream-based architectures thought its core feature is original.


## Introduction

Main idea is to decompose the system in components each with a single responsibility, that communicate with the minimum needed consistency and that can be deployed independently.

The system publishes an interface of Commands that change the state and of Queries that retrieves data; it uses polyglot, denormalized databases and asynchronous communication based on messages.

High performances are due to the original concept of the “Lock Table” that reduces as much as possible the “transactional boundaries” (see DDD) allowing transaction parallelization hard to match.

The following sections give an overview of the architectural components and a detailed description of the “Lock Table” concept. Then all components are described in depth and finally it’s explained how the platform achieves the set goals.


## Architectural overview of the components

The system allows two kinds of operations, **Command **and **Query, **(see CQRS); the earlier changes the state of the system, the latter returns data to the user and has no side effects.

A component named **Integrator** offers HTTP end-point for Commands and Queries. The Command object is validated by a component named **Validator** written for that kind of command and that is able of accepting and refusing it (see Invariant in DDD). Validators need one or more Projections to work, which will be explained later.

An accepted Command is then handled by the component **Command Store**, whose responsibility is to persist it (see Event Sourcing). From now on the Command is officially _happened_. Command Store persists the Command on a append-only database or filesystem optimized for this task; it uses a special data structure named **Lock Table** to minimize the effect of the sequential (non-parallel) Command computation; it guarantees system consistency and allows Commands parallel execution to a certain degree. Lock Table is described later in a dedicated section.

A component named **Actuator** builds, from the persisted Command, an object **Event** that describes the Command and its result: the Event is then spread in a publisher-subscriber fashion to the interested components that follow (see DDD).

Components named **Appliers**, subscribed to that kind of Event update the related **Projections**, persisting “views” of the data. They contain the data that the users want to see on request, in the form of tables, documents, byte arrays or others. The Applier updates its Projection according to the Event it received.

**Queries**, created by the Integrators following external requests, use the related Projection to return data to the requester; these Projections can be dedicated to a single Query type.

Updated according to the Events received by their Appliers, Query Projections are, by their own nature, “eventually consistent” compared to the “transactional consistency” of Command persistency; they can become “transactionally consistent” but with a reduced throughput.


## The Lock Table

It’s a table that implements a fine-grained lock system. It contains exactly three columns: lock id, read counter, write counter. 

Each record is related to a “resource”. The counters express how many times the resource was read or written, as a version identifier of the resource. Each record, named “lock” represents a transactional boundary on the resource(See DDD). Each lock is updated in the same transaction with the persistence of the Command in the Command Store.

Counters values are used to version Commands and Events: they both always contain the version number of the used Projections, that describe the state of the system when the Command was accepted.

In this way a specific lock table will always be able to recognize if a Command was accepted using up-to-date, consistent data and if not it will be able to request another Validation or refuse the Command. This allows the system to stay strictly, transactionally consistent, allowing eventually consistent changes of specific parts of the view model, the Projections. (See CQRS)

Using this simple mechanism it’s possible to compose various locks in lock tables to create transactional boundaries (see DDD) complex as needed; this allows to efficiently and easily tune performances of Command computation and of specific parts of the engine in terms of latency and throughput.

In the simplest case there’s one lock per Projection; for each Command there’s a lock table containing the locks of the Projection touched by the Command. This method allows natural bounded context division (see DDD) and shows points of contacts between Commands and related Projection, allowing aware integration of the two.

For example, if a Command was accepted by a Validator using Projection X with lock at 100, on such Command will be recorded “100 for Projection X”. If the system were not up-to-date and consistent due to eventual consistency, lock table would describe Projection X with lock value greater than 100, like 101: lock table, which is transactionally consistent with the Command Store, would be set at 101, but not the Projection X which still is set to 100. In this situation the Command that was based on outdated Projection X, with lock at 100, will be refused by the lock table.


## Other components


### Integrator

They are the adapters of “Hexagonal Architecture” pattern; their goal is to integrate external world and technology with the CIEPQ engine.


### Validator

They execute validation of Commands from a business point of view. They may use Projections for this task: the version identifier of the used Projections are used to update the related Lock Table. The Validated Command will contain the used version identifiers.


### Command Store

Its goal is to persist validated Commands accepted by Validators. It is the only source of truth about the state of the system and it guarantees data durability and integrity.


### Actuator

They convert, where needed, Commands into Events to optimize propagation of the state updates to the whole system.


### Event Store

Akin to the Command Store, but for Events; it is not a single source of truth, it raises resiliency, speed of recovery in case of fault, speed of launching new nodes.


### Applier

They receive Commands and/or Events in order to updated Projections and Triggers. If Commands/Events arrive in an unexpected order, easily detectable using lock tables or similar mechanisms, Applier will require missing Command/Events to the Command/Event Store.


### Projection

It accepts Query, which are requests for information about an observable part of the system.


### Trigger

On specific conditions it prepares and throws a new Command. Useful to decompose complex elaborations in multiple transactionally-independent parts that can run concurrently. (See Sagas in DDD)


## Distributed architecture

System decomposition in the previously described components allow to take fine grain decisions and easily tune handable load.

The Command Store is the only real bottleneck, where Commands computations are serialized. Before and after this component every computation can be made parallel and messages can be spread through a publish-subscribe mechanism. Tuning of latency/throughput ratio can be achieved by decomposing lock tables in smaller lock tables, faster to be verified. The same effect occurs on Validators.

All other components are distributable, assignable to dedicated machines, assuming eventual consistency is acceptable. Consistency between distributed components can be handled with message-based sagas (See DDD)

Commands are numbered with ids when they are saved in the Command Store. Being the only non-parallel point, there’s no risk to assign the same id to two different Commands. Each Projection is aware of the last ids that had an effect on it; it is then able to assess if it lagged behind (i.e. it lost Events due to network issues) and then signal the problem.


## Obtain resiliency: fault tolerance, disaster recovery

The only “master” database is the append-only Command Store; it does not support updates or deletes but only inserts at the end of the datastore. It is not usually used for reads.

Other databases, named conventionally Projections, are updated by Commands and are denormalized.

As said previously, the Projection has an Applier able to assess if the Projection lagged behind (network error, host restarted) and then not up-to-date: the Applier can update again the Projection by requesting Commands to the Command store from a certain Command id. A completely lost Projection (hard disk failure, flood,...) can be rebuilt from zero in the same way, reapplying all the Commands from the Command Store.

With this configuration the largest part of the resiliency investment is made on the Command Store; Projections can be replicated and delocalized at will, keeping the service up in case of failure of single nodes. In case of data loss, recovery can occur automatically.

Another optimization is to not read from the Command Store in order to not slow down the incoming Commands, using instead **Event Stores** for Projection recovery. An Event Store is an append-only database that allows Event reads from a certain Event-index. They are caches dedicated to fast rebuild of Projections and they are not master copies of data.
