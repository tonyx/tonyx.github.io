# Commands

(this section needs an update as we have a new )

A Command type can be represented by a Discriminated Union. Executing the command on a specific context or aggregate means returning a new state and, accordingly, a list of events, or an error.
You can also specify _"command undoers"_, that allow you to compensate the effect of a command. An undoer returns a new function that in the future can be executed to return the events that can reverse the effect of the command itself.
For example, the "under" of AddTodo is the related RemoveTodo (see next paragraph).

In the following code we can see the signature for any state viewers for any context or aggregate. State viewer corresponds to read models: they will provide the current state of aggregate or context. Typically, that state may come from a cache, from the event store (by processing the events) or from a topic of Kafa (or eventually any other message/event broker, even though I haven't implemented completed any of them yet).

```FSharp

    type StateViewer<'A> = unit -> Result<EventId * 'A, string>
    type AggregateViewer<'A> = Guid -> Result<EventId * 'A,string>
    
    type Aggregate<'F> =
        abstract member Id: Guid // use this one to be able to filter related events from same string
        abstract member Serialize: 'F
    
    type Event<'A> =
        abstract member Process: 'A -> Result<'A, string>

    type Command<'A, 'E when 'E :> Event<'A>> =
        abstract member Execute: 'A -> Result<'A * List<'E>, string>
        abstract member Undoer: Option<'A -> StateViewer<'A> -> Result<unit -> Result<List<'E>, string>, string>>
        
    type AggregateCommand<'A, 'E when 'E :> Event<'A>> =
        abstract member Execute: 'A -> Result<'A * List<'E>, string>
        abstract member Undoer: Option<'A -> AggregateViewer<'A> -> Result<unit -> Result<List<'E>, string>, string>>


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
Any command must ensure that it will return Result.OK (and therefore, one or more events) only if the events to be returned, when processed on the current state, return an Ok result, i.e. a valid state (and no error). 

There are two version of the _evolve_: one tolerates inconsistent events and another one will fail in case just an event will return an error.
The way the evens are stored in the event store ensures that no stored event will return inconsistent state when processed. Therefore future releases of the library will probably use by default the unforgiving version of the _evolve_ function.

## Undoer

Here is an example of the use of an undoer for a command:


```fsharp
module CartCommands =
    type CartCommands =
    | AddGood of Guid * int
    | RemoveGood of Guid
        interface AggregateCommand<Cart, CartEvents> with
            member this.Execute (cart: Cart) =
                match this with
                | AddGood (goodRef, quantity) -> 
                    cart.AddGood (goodRef, quantity)
                    |> Result.map (fun s -> (s, [GoodAdded (goodRef, quantity)]))
                | RemoveGood goodRef ->
                    cart.RemoveGood goodRef
                    |> Result.map (fun s -> (s, [GoodRemoved goodRef]))
            member this.Undoer = 
                match this with
                | AddGood (goodRef, _) -> 
                    Some 
                        (fun (cart: Cart) (viewer: AggregateViewer<Cart>) ->
                            result {
                                let! (i, _) = viewer (cart.Id) 
                                return
                                    fun () ->
                                        result {
                                            let! (j, state) = viewer (cart.Id)
                                            let! isGreater = 
                                                (j >= i)
                                                |> Result.ofBool (sprintf "execution undo state '%d' must be after the undo command state '%d'" j i)
                                            let result =
                                                state.RemoveGood goodRef
                                                |> Result.map (fun _ -> [GoodRemoved goodRef])
                                            return! result
                                        }
                                }
                        )
                | RemoveGood goodRef ->
                    Some
                        (fun (cart: Cart) (viewer: AggregateViewer<Cart>) ->
                            result {
                                let! (i, state) = viewer (cart.Id) 
                                let! goodQuantity = state.GetGoodAndQuantity goodRef
                                return
                                    fun () ->
                                        result {
                                            let! (j, state) = viewer (cart.Id)
                                            let! isGreater = 
                                                // this check depends also on the number of events generated by the command (i.e. the j >= (i+1) if command generates 2 event)
                                                (j >= i)
                                                |> Result.ofBool (sprintf "execution undo state '%d' must be after the undo command state '%d'" j i)
                                            let result =
                                                state.AddGood (goodRef, goodQuantity)
                                                |> Result.map (fun _ -> [GoodAdded (goodRef, goodQuantity)])
                                            return! result
                                        }
                                }
                        )
 ```


This is the abstract definition of an of the undoer of an aggregate.
```FSharp
        abstract member Undoer: Option<'A -> AggregateViewer<'A> -> Result<unit -> Result<List<'E>, string>, string>>
```

The meaning is:
Extract from the current state of the aggregate useful info for a future "rollback"/"undo" and return a function that, when applied to the current state of the aggregate, will return the events that will "undo" the effect of this command.

You may need undoers if the event store doesn't support multiple stream transactions or if you will use a distributed architecture with many nodes handling different streams of events.
By using PostgresSQL as event store you can just set the undoer to None as the event store will handle the cross-streams transactions for us. 

[Commands.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/Domain/Tags/Commands.fs)


