# Commands

Commands are based on Discriminated Union types and, when executed, will produce lists of events or error.
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
Commands can use event caching if it is enabled as we have seen in the previous section.

## Undoer

A command may have associated an _undoer_ which is similar to a command but it is aimedo to compensate the effect of a command in case such command is a part of a multiple stream transaction that fails. We need the _undoer_ only if the storage lack of multiple streams transactions (which is the case of EventStoreDb)

```FSharp
    type Undoer<'A, 'E when 'E :> Event<'A>> = 'A -> Result<List<'E>, string>
```


I use the Undoer in the experimental "LightRepository" and EventStoreBridge.
So, as a recap: when the repository uses only in memory or Postgres storage, it has no need of any undoer because in memory and postgres storage support multiple stream transactions.
If you will use EventStoreDb as storage, you will need to provide an undoer for each command that is part of a multiple stream transaction.

Before giving an example of undoer let me rewrite the abstract definition of a command.

```FSharp
    type Command<'A, 'E when 'E :> Event<'A>> =
        abstract member Execute: 'A -> Result<List<'E>, string>
        abstract member Undoer: Option<'A -> Result<Undoer<'A, 'E>, string>>
```

This is the how the Undoer of a command works:  given the current state of the aggregate, it returns a function that applied to the states actually returns an undoer.
In this way the undoer is ready to be executed and can return a list of events that can compensate the effect of the command.

So the undoers works in two shots: one to build a context for the eventual future undo, and one to atually do the "undo". 

Here is an example:

The removeTag command returns a list of TagRemoved events. When they are executed they will remove the tag from the aggregate state.

In a transaction context if you want to be ready to eventually reverse the effect of the removeTag command you need to readd the tag to the aggregate state. For this reason you need to build a context before the removal so that the tag to be readded is actually available.

From the code perspective:

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


