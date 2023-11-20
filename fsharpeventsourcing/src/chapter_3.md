# Events

Events are discriminated unions (DU) with cases associated with members of the cluster doing _add_/_update_/_delete_/ entities.

When we process an event it returns a new state or an error, which fits the following definition of Process taken from the Core.fs file:

```FSharp
    type Event<'A> =
        abstract member Process: 'A -> Result<'A, string>
```
The _'A_ is the generic cluster type the event is associated with.

This is an example of a concrete implementation of an event related to the TodoCluster members _Add_ and _Remove_.

So, for example, the _TodoAdded_ event is associated with the _AddTodo_ member.
The _Process_ member of the event is implemented by calling the related clusters member.

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

