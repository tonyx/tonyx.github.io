# Testing

There are plenty of tests in the examples folders.
 of the Sharpino repository.
the structure is based on using a setup that wipe the event store and the cache before each test case. The test must be run in a sequence to avoid interference between them because of the state of the event store and the cache.o

An extension of Expecto provide a "multipleTestCase` function that makes the test parametric.

A particular case of using the _multipleTestCase_ is to run the same test case with different event store implementations (i.e. inmemory, Postgres json, Postgres binary eventually in conjuction with Rabbitmq).

A particular parameter needed to test by using RabbitMQ as event broker is to run the tests in sequence with some delay to allow the propagation of the messages between the different consumers. The delay parameter will so depend on which configuration is under test.

An example of setting "instances" (which should mean 'parameters') to feed the test cases is the following:

```FSharp
let instances =
    [
        #if RABBITMQ
            (fun () -> setUp pgEventStore),  ItemManager(pgEventStore, rabbitMqItemStateViewer, rabbitMqReservationStateViewer, messageSenders), 100
        #else
            (fun () -> setUp(pgEventStore)), ItemManager(pgEventStore, pgStorageItemViewer, pgStorageReservationViewer), 0
            (fun () -> setUp(memEventStore)),  ItemManager(memEventStore, memoryStorageItemViewer, memoryStorageReservationViewer), 0
        #endif
    ]
```

That configuration use two (or three if Rabbitmq is defined) different event stores and viewers to run the same test cases.

In the examples we see that there exist "viewers" based on the event store and based on Rabbitmq consumers.

Any "viewer" based on the eventStore (which uses also the cache in between) has the following signature:

```FSharp
    type AggregateViewer<'A> = AggregateId -> Result<EventId * 'A,string>
```
By using a viewer based on the event store we can rely on this function present 
in the command handler:

```FSharp
    let inline getAggregateStorageFreshStateViewer<'A, 'E, 'F
        when 'A :> Aggregate<'F> 
        and 'A : (static member Deserialize: 'F -> Result<'A, string>) 
        and 'A : (static member StorageName: string) 
        and 'A : (static member Version: string) 
        and 'E :> Event<'A>
        and 'E: (static member Deserialize: 'F -> Result<'E, string>)
        >
        (eventStore: IEventStore<'F>) 
        =
            fun (id: Guid) ->
                result
                    {
                        let! (eventId, result) = getAggregateFreshState<'A, 'E, 'F> id eventStore
                        return
                            (eventId, result :?> 'A) 
                    }
```
Note: a critical point is that given that the cache layer is type agnostic, some cast/boxing is needed to return the proper type of the aggregate. It seems that this use is safe enough as long as the following constraints is satisfied:
whenever an aggregate of type 'A is stored in the event store, it is retrieved as type 'A as well. This aspect should be invisible to the application using the library but, still, it worth mentioning it.
Also note that the box/unboxing operations may have a performance impact, but that is overcomed by the empirical evidence of the improvement given by the cache layer.

A viewer based on Rabbitmq deserves a different chapter. However the provided examples are based on using some blueprint to facilitate handling the aggregate states of a certains stream as a ConcurrentDictionary:

```FSharp
    ConcurrentDictionary<AggregateId, (EventId * 'A)

```

Many examples based on RabbitMq use such blueprints to implement a "Consumer" thet, among all, provide a viewer function that can be used in the service layer.

```FSharp
        member this.GetAggregateState (id: AggregateId) =
            if (statePerAggregate.ContainsKey id) then
                statePerAggregate.[id]
                |> Result.Ok
            else
                Result.Error "No state" 

```

In essence: by instrumenting any stream with RabbitMq consumer, and by also instrumenting the tests to launch the consumers as service, it will be possible to use and test the viewers based on RabbitMq.









