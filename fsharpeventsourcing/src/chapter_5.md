# Application service layer

An application service layer provides services that use Repository and Storage to get the state and/or send commands to one or more aggregates (eventually in an atomic/transactional way with potential performance issues) and store the related events.

Here is one of the simplest examples of an entry for a service involving a single aggregate, by building and running an AddTag command.


```FSharp
    member this.addTag tag =
        result {
            let! _ =
                tag
                |> AddTag
                |> (runCommand<TagsAggregate, TagEvent> storage)
            return ()
        }
```

The service layer sends commands to the repository so that this one can run it producing and storing the related events.

The following example shows a service layer that uses two aggregates and an explicit lock (note that the lock object concept to handle transactions has been substituted by a mailboxprocessor (actor model). Still I'm not sure if the lock object approach deserved to be dismissed).
As mentioned in the previous section its not a big deal avoiding any locking or mailboxprocessor (single thread) command processing: the worst that may happen is that the events stored are inconsistent and will be skipped by the "evolve" function.

In the following case I assume that the aggregates where using lock objects to handle transactions (which is not the case anymore).

```FSharp
    member this.addTodo todo =
        lock TagsAggregate.LockObj <| fun () ->
            result {
                let! (_, tagState) = getState<TagsAggregate, TagEvent>(storage)
                let tagIds = tagState.GetTags() |>> (fun x -> x.Id)

                let! tagIdIsValid =    
                    (todo.TagIds.IsEmpty ||
                    todo.TagIds |> List.forall (fun x -> (tagIds |> List.contains x)))
                    |> boolToResult "A tag reference contained is in the todo is related to a tag that does not exist"

                return! 
                    todo
                    |> TodoCommand.AddTodo
                    |> (runCommand<TodosAggregate, TodoEvent> storage)
            }
```

The todo can be added only if it contains valid tag references.

Now I am showing how I decided to deal with the same issue of making the code safe  (transactional) by using the mailboxprocessor, part of FSharp core library, that allows processing messages in a single thread.

```FSharp
        member this.AddTodo todo =
            let f = fun () ->
                ResultCE.result {
                    let! (_, tagState) = storage |> getState<TagsAggregate, TagEvent> 
                    let tagIds = tagState.GetTags() |>> (fun x -> x.Id)

                    let! tagIdIsValid =    
                        (todo.TagIds.IsEmpty ||
                        todo.TagIds |> List.forall (fun x -> (tagIds |> List.contains x)))
                        |> boolToResult "A tag reference contained in the todo is related to a tag that does not exist"

                    let! _ =
                        todo
                        |> TodoCommand.AddTodo
                        |> runCommand<TodosAggregate, TodoEvent> storage
                    let _ = 
                        storage
                        |> mkSnapshotIfInterval<TodosAggregate, TodoEvent>
                return ()
            }
            async {
                return processor.PostAndReply (fun rc -> f, rc)
            }
            |> Async.RunSynchronously
```

The entire expression is wrapped in an async block and the processor.PostAndReply function is used to send the function f to the processor and wait for the result (again: the processor is a single thread message processor that may not be required anymore).

Basically this approach ensures single-thread processing but may slow down the processing of commands if the aggregate is involved in a long-running transaction.

Another example is the following, about sending commands to more aggregates.
This code removes the tag with any reference to it. It builds two commands and makes the repository process them at the same time.
This code removes a tag and any reference to it.

```FSharp
    member this.removeTag id =
        ResultCE.result {
            let removeTag = TagCommand.RemoveTag id
            let removeTagRef = TodoCommand.RemoveTagRef id
            let! _ = runTwoCommands<TagsAggregate, TodosAggregate, TagEvent, TodoEvent> storage removeTag removeTagRef
            return ()
        }
```

About to the example involving the "under" of a command: the runTwoCommands function is executed in a transactional context if the storage supports multiple streams transactions (i.e. Postgres or in memory).
In the case the storage does not support multiple streams transactions (i.e. Eventstoredb) the runTwoCommands function will execute the two commands in a sequence and use the undoers (if provided) to rollback the effect of the first command in case the second command fails.

Source: [App.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/App.fs)















