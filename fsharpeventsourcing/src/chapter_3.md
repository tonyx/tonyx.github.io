# Events

As I said before, some members of an aggregate act virtually as "state changers", operating add/update/delete on one or more of its models. 
Each aggregate has events and commands as discriminated unions (DU) with cases associated to those members.

An event is something that, when processed, returns a new state of the aggregate or an error, which fits the following definition taken from the Core.fs file:

```FSharp
    type Event<'A> =
        abstract member Process: 'A -> Result<'A, string>
```
The _'A_ is the generic aggregate type the event is associated with.

This is an example of a concrete implementation of an event related to the TodoAggregate members _Add_ and _Remove_.
Particularly, the _TodoAdded_ event is associated to the _AddTodo_ member and the _TodoRemoved_ event is associated to the _RemoveTodo_ member and the _Process_ member of the event is implemented by calling the related aggregate member.

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
You may want to do do event caching by wrapping the call to the specific aggregate member in a lambda expression without evaluating it and passing it to a specific cache manager that may eventually evaluate it and return the result or the cached result if the event has already been processed.

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

Having said that about caching, I wanrn that caching events don't look so much like a "performance booster". It looks more promising the caching of the state of the aggregate (see the next section).

Therefore event Caching is disabled by default. See "EVENTS_CACHING_ENABLED" in the project file: [project file](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/Sharpino.Lib.fsproj)
 To enable the caching of events, the library must be compiled with the following compilation symbol: `EVENTS_CACHING_ENABLED`.
 (unfortunately, at the moment - verson 1.0.1, you cannot change it if you add the library from nugt. You need to clone the repo and compile it yourself).

Source code:  [Events.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/aggregates/Todos/Events.fs)

