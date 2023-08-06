# Commands

Commands are based on DU types and, when executed, will produce lists of events or error.
The abstract definitions of Command and Undoer (see later) are:

```FSharp

    type Undoer<'A, 'E when 'E :> Event<'A>> = 'A -> Result<List<'E>, string>

    type Command<'A, 'E when 'E :> Event<'A>> =
        abstract member Execute: 'A -> Result<List<'E>, string>
        abstract member Undoer: Option<'A -> Result<Undoer<'A, 'E>, string>>

```

Example:

```FSharp
    type TodoCommand =
        | AddTodo of Todo
        | RemoveTodo of Guid

        interface Command<TodosAggregate, TodoEvent> with
            member this.Execute (x: TodosAggregate) =
                match this with
                | AddTodo t -> 
                    match x.AddTodo t with
                    | Ok _ -> [TodoAdded] |> Ok
                    | Error x -> x |> Error
                | RemoveTodo g ->
                    match
                        x.RemoveTodo g with
                        | Ok _ -> [TodoEvent.TodoRemoved g] |> Ok
                        | Error x -> x |> Error
            member this.Undoer = None
```

Commands return events in a list, so they can return multiple events, as follows:

```FSharp
    type TodoCommand =
        [...]
        | Add2Todos of Todo * Todo
        interface Command<TodosAggregate, TodoEvent> with
            member this.Execute (x: TodosAggregate) =
            [...]
            match this with
            | Add2Todos (t1, t2) -> 
                let evolved =
                    fun () ->
                    [TodoEvent.TodoAdded t1; TodoEvent.TodoAdded t2]
                    |> evolve x
                match evolved() with
                    | Ok _ -> [TodoEvent.TodoAdded t1; TodoEvent.TodoAdded t2] |> Ok
                    | Error x -> x |> Error
            member this.Undoer

```
Any command must ensure that it returns some events only if those events, when applied to the current aggregate state, will return an Ok result (and no error). 
Commands can use event caching if it is enabled.

## Undoer

A command may have associated an _undoer_ which is similar to a command but it is aimedo to  compensate the effect of a command in case such command is a part of a transaction that fails. We need the _undoer_ only if the storage lack of multiple streams transactions.

```FSharp
    type Undoer<'A, 'E when 'E :> Event<'A>> = 'A -> Result<List<'E>, string>
```


I use the Undoer in the experimental "LightRepository" and EventStoreBridge.
So, as a recap: ordinary repository that uses only in memory or Postgres storage will not need the undoer because in memory and postgres storage support multiple stream transactions.

Before giving an example of undoer let me rewrite the abstract definition of a command.

```FSharp
    type Command<'A, 'E when 'E :> Event<'A>> =
        abstract member Execute: 'A -> Result<List<'E>, string>
        abstract member Undoer: Option<'A -> Result<Undoer<'A, 'E>, string>>
```

The optional member Undoer of a command, given the current state of the aggregate, returns a function that applied to the states returns an undoer.
Then the undoer is ready to be executed and can  return a list of events that can compensate the effect of the command.

In the following example I try to explain why the undoer needs to work in two shots: at first it taks as parameter the state of the aggregate before any operation involving it in a transaction returning a new function to be executed eventually later.
If the transaction fails then the function returned can be applied to reverse the effect of the already executed command.


The removeTag command returns a list of TagRemoved events. When they are executed they will remove the tag from the aggregate state.

In a transaction context if you want to reverse the effect of the removeTag command you need to readd the tag to the aggregate state. For this reason you need to build a context before the removal.

This is the example of the undoer for the tags:

```Fsharp
    member this.Undoer = 
        match this with
        | RemoveTag g -> 
            // block to be executed before the actual command removing tag is executed. It will return another function with the context needed (the tag itself)
            (fun (x: TagsAggregate) ->
                result {
                    let! tag = x.GetTag g
                    let result =

                        // block to be executed after the actual command removing tag is executed. It will return the list of events to be applied to the aggregate state to compensate the effect of the command. Note that the tag is the context needed to readd the tag to the aggregate state.

                        fun (x': TagsAggregate) ->
                            x'.AddTag tag 
                            |> Result.map (fun _ -> [TagAdded tag])
                    return result
                }
            )
            |> Some
        | AddTag t ->
            // this case is simple than the previous because there is no need to retrieve anything from the context before the command is executed. The context is the tag itself (particularly its id), that is not lost during the transaction.
            (fun (_: TagsAggregate) ->
                fun (x': TagsAggregate) ->
                    x'.RemoveTag t.Id 
                    |> Result.map (fun _ -> [TagAdded t])
                |> Ok
            )
            |> Some

```



Sources: [Commands.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/aggregates/Todos/Commands.fs))


