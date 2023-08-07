# Commands

There is a concrete implementation of command for each aggregate.
We define Commands as Discriminated Union type and, when executed, they will produce a lists of events or an error.
We have also _"command undoers"_, that allow us to compensate the effect of a command in case it is part of a multiple stream transaction that fails  as we will see later.

The abstract definitions of Command and Undoer  are:

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

It is unusual to define commands such as they will return multiple events, but we can do it, as follows:

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
                    |> evolveUNforgivingErrors x
                match evolved() with
                    | Ok _ -> [TodoEvent.TodoAdded t1; TodoEvent.TodoAdded t2] |> Ok
                    | Error x -> x |> Error
            member this.Undoer

```
It is important that any command must ensure that it returns some events only if those events, when applied to the current aggregate state, will return an Ok result (and no error). 
For that reason, before returning the events, I invoked the "evolveUNforgivingErrors" function to probe the sequence of two events to be eventually returned. The evolveUNforgivingErrors applies the events to the current state and will return an error if the result is an error.
The ordinary evolve is able to skip errors and return the state anyway.
Commands can use event caching if it is enabled as we have seen in the previous section.

## Undoer

A command may have associated an _undoer_ which is similar to a command itself but it is aimed to eventually do the "reverse" of a command, which means  compensating the effect of a command in case such command is a part of a multiple stream transaction that fails. We need the _undoer_ only if the storage lack of multiple streams transactions (which is the case of EventStoreDb)

```FSharp
    type Undoer<'A, 'E when 'E :> Event<'A>> = 'A -> Result<List<'E>, string>
```


So, as a recap: when the repository uses only in memory or Postgres storage, it doesn't need any _undoer_ because in memory and Postgres storage support multiple stream transactions.
If you will use an EventStoreDb like engine as storage, you may want to provide an undoer for each command in case it will be part of a multiple stream transaction.

I am going to show an example of an undoer but before that let me remind the abstract definition of a command.

```FSharp
    type Command<'A, 'E when 'E :> Event<'A>> =
        abstract member Execute: 'A -> Result<List<'E>, string>
        abstract member Undoer: Option<'A -> Result<Undoer<'A, 'E>, string>>
```

This shows that the command may provide an Undoer which is a function that returns an actual undoer (!).
Here is the way we can user an under in a command: given the current state of the aggregate, the "command undoer" returns a function that, applied to the state, actually returns an actual undoer.
So the actual undoer now available is ready to be executed and can return a list of events that can compensate the effect of the command if we need it.

So the undoers works in two shots: one to build a context for the eventual future undo, and one to atually... _do_ the _undo!_. 

Concretely:

The removeTag command returns a list of TagRemoved events. When those events are processed the result is the aggregate without the tags.

In a transaction context if you want to be ready to eventually reverse the effect of the removeTag command you need to readd the tag to the aggregate state. For this reason you need to build a context before the removal so that the tag to be eventually readded is still available.

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


