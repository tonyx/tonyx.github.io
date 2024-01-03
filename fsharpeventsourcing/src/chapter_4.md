# Commands

A Command type is a Discriminated Union. Executing the command on a specific cluster means returning a proper list of events or an error.
You can also specify _"command undoers"_, that allow you to compensate the effect of a command in case it is part of a multiple stream transaction that fails as we will see later. An undoer issues the events that can reverse the effect of the related command.
For example, the "under" of AddTodo is the related RemoveTodo (see next paragraph).


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

        interface Command<TodosContext, TodoEvent> with
            member this.Execute (x: TodosContext) =
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

A command may return more than one event:

```FSharp
    type TodoCommand =
        [...]
        | Add2Todos of Todo * Todo
        interface Command<TodosCluster, TodoEvent> with
            member this.Execute (x: TodosContext) =
            [...]
            match this with
            | Add2Todos (t1, t2) -> 
                let evolved =
                    fun () ->
                    [TodoEvent.TodoAdded t1; TodoEvent.TodoAdded t2]
                    |> evolveUNforgivingErrors x
                match evolved() with
                    | Ok _ -> [TodoEvent.TodoAdded t1; TodoEvent.TodoAdded t2] |> Ok
                    | Error x -> x |> Error
            member this.Undoer

```
Any command must ensure that it will return Result.Ok (and therefore, one or more events) only if the events to be returned, when processed on the current state, return an Ok result, i.e. a valid state (and no error). 

The _evolve_ tolerates inconsistent events.
Thus the _evolve_ will just skip events that, when processed, return an error.

## Undoer

A command case may have associated an _undoer_ which is similar to a command itself and is aimed to eventually do the "reverse" of that command case.
We need the _undoer_ only if the storage we use lacks support for multiple stream transactions (like EventStoreDb).
(Note: I may not support EventStoreDb in the future. I probably will just stick to the Postgres Db as event-store and Apache Kafka as event-broker.)

```FSharp
    type Undoer<'A, 'E when 'E :> Event<'A>> = 'A -> Result<List<'E>, string>
```

When we use storages like _in memory_ or _Postgres_, we don't need to define any _undoer_ for our commands, because those storages support multiple stream transactions.
Otherwise, we need to define an _undoer_ if we know we are going to pass them to _runTwoCommands_ on the command handler.

The abstract definition of a command is:
```FSharp
    type Command<'A, 'E when 'E :> Event<'A>> =
        abstract member Execute: 'A -> Result<List<'E>, string>
        abstract member Undoer: Option<'A -> Result<Undoer<'A, 'E>, string>>
```

The "command undoer" returns a function that, applied to the state, must return the actual _undoer_.

So the undoers work in two shots: one to build a context for the eventual future undo, and one to actually... _do_ the _undo_. 

__Example of "undoer"__ :

The _RemoveTag_ command returns a list of TagRemoved events. We know that when those events are processed the result is the new cluster state without the tags.
However, we may want to roll back the effect of those events by adding events that reverse their effect.

This applies to a transaction context: you need to be able to re-add the tag to the state. For this reason, you need to build a context _before_ the removal so that the tag to be eventually readded is still available.

From the code with some comments:

```Fsharp
    member this.Undoer = 
        match this with
        | RemoveTag g -> 
            // block to be executed before the actual command removing tag 
            // is executed. It will return another function with the context needed (the tag itself)
            (fun (x: TagsContext) ->
                result {
                    let! tag = x.GetTag g
                    let result =

                        // block to be executed after the actual command removing tag is executed.  
                        // It will return the list of events to be applied to the cluster state to compensate the effect of the command. 
                        // Note that the tag is the context needed to readd the tag to the  state.

                        fun (x': TagsContext) ->
                            x'.AddTag tag 
                            |> Result.map (fun _ -> [TagAdded tag])
                    return result
                }
            )
            |> Some
        | AddTag t ->
            // this case is simple than the previous because there is no need to retrieve anything from the context before the command is executed. 
            // The context is the tag itself (particularly its id), that can't be lost during the transaction.
            (fun (_: TagsContext) ->
                fun (x': TagsContext) ->
                    x'.RemoveTag t.Id 
                    |> Result.map (fun _ -> [TagAdded t])
                |> Ok
            )
            |> Some

```


[Commands.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/Domain/Tags/Commands.fs)


