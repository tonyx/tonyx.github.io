# Events

As I said before, some members of an aggregate act as "state changer", operating add/update/delete on one or more of its models. 
Each aggregate has an associated Event type and such event type is a Discriminated Union (Du) type with a case associated to "state changer" members of the aggregate.

An event type can be defined, in abstract, as something that, when processed, return a new state of the aggregate or an error, which fits the following definition:

```FSharp
    type Event<'A> =
        abstract member Process: 'A -> Result<'A, string>
```
The _'A_ is the generic aggregate type the event is associated to.

This is an example of a concrete implementation ov event relate to the TodoAggregate members _Add_ and _Remove_:

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

You can do event caching by wrapping the call to the specific aggregate member in a lambda expression without evaluating it, and passing it to a specific cache manager that may eventually evaluate it and return the result or the cached result if the event has already been processed.

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

Having said that about caching, I am warning that  caching events doesn't look so much a "performance booster". It looks more promising the caching of the state of the aggregate (see the next section).

Therefore event Caching is disabled by default. See "EVENTS_CACHING_ENABLED" in the project file: [project file](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/Sharpino.Lib.fsproj)
 To enabled the caching of events, the library must be compiled with the following compilation symbol: `EVENTS_CACHING_ENABLED`.

Source:  [Events.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/aggregates/Todos/Events.fs)

