# Events

As I said before, some instance members of a cluster act virtually as "state changers", operating add/update/delete on some of the entities that it handles.
Events and commands are  discriminated unions (DU) with cases associated to those members.

An event is something that, when processed, returns a new state or an error, which fits the following definition taken from the Core.fs file:

```FSharp
    type Event<'A> =
        abstract member Process: 'A -> Result<'A, string>
```
The _'A_ is the generic cluster type the event is associated with.

This is an example of a concrete implementation of an event related to the TodoCluster members _Add_ and _Remove_.
Particularly, the _TodoAdded_ event is associated to the _AddTodo_ member and the _TodoRemoved_ event is associated to the _RemoveTodo_ member and the _Process_ member of the event is implemented by calling the related clusters member.

```Fsharp
    type TodoEvent =
        | TodoAdded of Todo
        | TodoRemoved of Guid
            interface Event<TodosCluster> with
                member this.Process (x: TodosCluster ) =
                    match this with
                    | TodoAdded (t: Todo) -> 
                        x.AddTodo t
                    | TodoRemoved (g: Guid) -> 
                        x.RemoveTodo g

```

Source code:  [Events.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/aggregates/Todos/Events.fs)

