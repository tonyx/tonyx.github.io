# Commands

(this section needs an update as we have a new )

A Command type is a Discriminated Union. Executing the command on a specific cluster or aggregate means returning a proper list of events or an error.
You can also specify _"command undoers"_, that allow you to compensate the effect of a command in case it is part of a multiple stream transaction that fails as we will see later. An undoer issues the events that can reverse the effect of the related command.
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

The _evolve_ tolerates inconsistent events.
Thus the _evolve_ will just skip events that, when processed, return an error.
This feature is associated with a specific permissive optimistic lock type that we will see later.

## Undoer

The use of the lambda expression is a nice trick for the undoers (the _under_ is returned as a lambda that retrieves the context for applying the undo and returns another lambda that actually can "undo" the command).

I haven't simplified any example of undoer yet, but this is the idea related to commands for adding and removing items 


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


Probably to follow the example its worth reading again the definition of an undoer:
```FSharp
        abstract member Undoer: Option<'A -> AggregateViewer<'A> -> Result<unit -> Result<List<'E>, string>, string>>
```

This can be simplified as the state of the aggregate is accessed in different ways, however this is the meaning:

Extract from the current state of the aggregate useful info for a future "rollback"/"undo" and return a function that, when applied to the current state of the aggregate, will return the events that will "undo" the effect of the command.

Saga like transaction handling will probably need this logic.
However, most of the time the commands between multiple aggregates uses db transactions (like Postgres) and the undoer is not needed.




[Commands.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/Domain/Tags/Commands.fs)


