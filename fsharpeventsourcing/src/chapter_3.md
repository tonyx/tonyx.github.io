# Events

Events are discriminated unions (DU) with cases associated with members of the context that end up in _adding_/_updating_/_deleting_/ entities.

When we process an event it returns a new state or an error: 

```FSharp
    type Event<'A> =
        abstract member Process: 'A -> Result<'A, string>
```
The _'A_ is the generic context or aggregate type, the event is associated with.

This is an example of a concrete implementation of an event related to the TodoCluster members _Add_ and _Remove_.

So, for example, the _TodoAdded_ event is associated with the _AddTodo_ member.
The _Process_ member of the event is implemented by calling the related clusters member.

```Fsharp
    type TodoEvent =
        | TodoAdded of Todo
        | TodoRemoved of Guid
            interface Event<TodosCluster> with
                member this.Process (x: TodosContext ) =
                    match this with
                    | TodoAdded (t: Todo) -> 
                        x.AddTodo t
                    | TodoRemoved (g: Guid) -> 
                        x.RemoveTodo g

```

Source code:  [Events.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/Domain/Todos/Events.fs)


