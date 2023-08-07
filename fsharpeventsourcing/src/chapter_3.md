# Events

Events are associated to the members of the aggregate that virtually change the state of the aggregate (i.e. doing virtual add/update/delete on their models).  They are defined as Discriminated Union (Du) type. 

The abstract definition of an Event is: 

```FSharp
    type Event<'A> =
        abstract member Process: 'A -> Result<'A, string>
```

Example of implementation relate to the TodoAggregate: add and remove:

```Fsharp
    type TodoEvent =
        | TodoAdded of Todo
        | TodoRemoved of Guid
            interface Event<TodosAggregate> with
                member this.Process (x: TodosAggregate ) =
                    match this with
                    | TodoAdded (t: Todo) -> 
                        x.AddTodo t
                    | TodoRemoved (g: Guid) -> 
                        x.RemoveTodo g

```
This shows that processing an event means calling the related aggregate member.

you can cache events by enabling them explicitly at a library level: [project file](Micro_ES_FSharp_Lib/Micro_ES_FSharp_Lib.fsproj):

```Fsharp
    type TodoEvent =
        | TodoAdded of Todo
        | TodoRemoved of Guid
            interface Event<TodosAggregate> with
                member this.Process (x: TodosAggregate ) =
                    match this with
                    | TodoAdded (t: Todo) -> 
                        EventCache<TodosAggregate>.Instance.Memoize (fun () -> x.AddTodo t) (x, [TodoAdded t]) 
                    | TodoRemoved (g: Guid) -> 
                        EventCache<TodosAggregate>.Instance.Memoize (fun () -> x.RemoveTodo g) (x, [TodoRemoved t]) 
```

Caching events does not show so much improvement so far.

Therefore event Caching is disabled by default. See "EVENTS_CACHING_ENABLED" in the project file: [project file](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/Sharpino.Lib.fsproj)
 To enabled the caching of events, the library must be compiled with the following compilation symbol: `EVENTS_CACHING_ENABLED`.

Source:  [Events.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/aggregates/Todos/Events.fs)

