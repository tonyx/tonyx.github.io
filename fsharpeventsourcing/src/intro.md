# Introduction

[Sharpino](https://github.com/tonyx/Micro_ES_FSharp_Lib) is a little F# event sourcing framework.
Two types of event sourced objects are supported: _Contexts_ and _Aggregates_.
_Contexts_: There is no specific id for a context so we just assume that a single instance and a single zero (initial state) instance exist for any _Context_.

Note: "Contexts"  will be not used anymore. A plain aggregate with a specific constand Id can substitute it (as in example 15). From here only "aggregates" (event sourced objects with an Id) will be mentioned.

_Aggregates_: Many instances of an aggregate type can exist at a time so we need an Id for each instance represented by a Guid. The initial state for any aggregate is its initial snapshot in the event store.

An aggregate implements at least one transformation members that will return a new instance of it or an error (using the Result data type) i.e. it returns ` Result<A, string>` 
An aggregate must also specify as static members ways to serialize and deserialize and the stream name and the snapshot interval. Examples of aggregates are given in the sample applications.
An aggregate must implement the _Aggregate_ interface based on a generic type that can be string or byte[] depending on the serialization method used. Json serialization will use string while binary serialization will use byte[].

Transformative members can be associated with events that, when processed, will return a new instance of the aggregate with a new state.
Commands are functions that given the current state of an aggregate and some parameters will return a pair of a new state of the aggregate and the list of events that, processed, will return such state.

Note: Beware that there are a specific commad type for Aggregates which is called AggregateCommand. See the definition in the source code.

Commands and events can be easily implemented using Discriminated Union types implementing the _Command_ and the _Event_ interfaces respectively.

Note: events must be serializable and deserialisable. Those functions use a generic serialization type (string or byte[]) in the same way as for aggregates.

Important note: many examples use the cross-aggregate transaction feature that has some advantages by allowing bidirectional references between aggregates making the navigation easier from any side. However, by adopting an in memory materialized view approach (see example 15) the need for cross-aggregate transactions is greatly reduced as the "navigation" can be done using the materialized view (or detailed view).

This document is a small guide to the Sharpino library (note: I cannot ensure that this is always up to date, read the examples starting from the end).
Focus on the following topics:

## The sample application 1.

This application will be deprecated as it fails in showing the main features of the library.

## Sample application 2. Booking system for seats in a stadium
Contexts represent rows. Some constraints are applied to the rows. The context is responsible for the seats. The seats are associated with the rows.
Note: there is a better way to implement this application using aggregates for the rows (see application 3).

## Sample application 3. Booking system for seats in a stadium
The same as application 2 but where rows are aggregates (so I can have any number of instances of seat rows). 

## Sample application 6. Pub system
Objects: Dishes, Ingredients, and Suppliers.

## Sample application 7. Shopping cart 
Objects Shopping Cart, Goods, a container with references to the existing goods. Note: the container ("context") can be ditched. the applications has two different versions: one using binary serialization and another using text/json serialization

## Sample application 8. Tycoon Transport
Partial implementation of the problem described here:
[Transport Tycoon](https://github.com/trustbit/exercises/blob/master/transport-tycoon-1.md)
Note: this will be revied by moving from bidirectional references to unidirectional references and materialized views/details. 

## Sample application 9. Classes, Teachers, Students, Reservations, Items

Classes, techer, students. Introducing the problem of course creation and
cancellation fees showing transactions on multiple objects. (i.e. course, students, teachers, balance...).
An experimental feature is related to "cross aggregates constraints" passed
as lambda expressions to a command: a command may query the state of objects
that are not directly related to the command. 

## Sample application 10.  Multiple commands of any type 

Thi example shows a way to execute multiple commands of any type in a single transaction. 
Note: it not advised to abuse this feature as a proper design is better and safer in using transaction scoped to a single aggregate type. However, in some cases this may be useful and is a fair alterative to the user of "sagas" or
"process managers" or "orchestrators" or "compensators" (which are supported anyway on the command side).


## Sample application 11. Students and Courses. Some performances meausurements

The examples check the performances by a creating massive number of students and courses.
If RabbitMQ is installed and running it is possible to use it as event bus to decouple the event store from the read model update process.

To check with RabbitMQ just run with the following command line:
`dotnet run --configuration:RabbitMQ`

## Sample application 12. Use of binary serialization

The serialization library provided with the framework is based on FsPiclker and supports both binary and json (even though any library can be used on the application side). This time binary is used.

## Sample application 13. Reservation pattern
In theory constraints can be voided by concurrent commands despite the use of optimistic concurrency control.
To face this possibility a reservation pattern can be used.

## Sample application 14. 
Using FSharp.SystemTextJson for serialization/deserialization instead of the built in json serializer.
Using type Id for any aggregate instead of Guid (primitive obsession).
By facing primitive obsession it is possible to get better type safety and avoid errors due to wrong Ids usage.

## Sample application 15. 
Use of details (materialized views) instead of cross-aggregate transactions.
Using refreshable details to  make the details able to be cached and refreshed when needed, i.e. when any event related to the streams that it depends on is committed.

General note: more often than note the examples can be executed using rabbitmq message sending.
Check the .fsproj files and if the RabbitMQ configuration is present it is possible to run the example using RabbitMQ as event bus by the command line below:
`dotnet run --configuration:RabbitMQ`