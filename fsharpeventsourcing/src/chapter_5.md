# Application service layer

An application service layer provides services that actually uses Respository and Storage to get the state and/or send commands to one or more aggregates eventyally in a coordinated (i.e. transactional) way.

Here is one of the simplest example of an entry for a services involving a single aggregate, by builing and running a AddTag command.


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

The service layer sends commands to the repository so that it can produce and store related events.

The following example shows a service function that uses two aggregates and an explicit lock (note the lock object concept to handle transaction has been substituted by a mailboxprocessor based actor model).

I will show an example involving the new version right after this one.

```FSharp
    member this.addTodo todo =
        lock TagsAggregate.LockObj <| fun () ->
            ResultCE.result {
                let! (_, tagState) = getState<TagsAggregate, TagEvent>(storage)
                let tagIds = tagState.GetTags() |>> (fun x -> x.Id)

                let! tagIdIsValid =    
                    (todo.TagIds.IsEmpty ||
                    todo.TagIds |> List.forall (fun x -> (tagIds |> List.contains x)))
                    |> boolToResult "A tag reference contained is in the todo is related to a tag that does not exist"

                let! _ =
                    todo
                    |> TodoCommand.AddTodo
                    |> (runCommand<TodosAggregate, TodoEvent> storage)
            return ()
        }
```

The todo can be added only if it contains valid tag references.

Now I am showing how I decided to deal with the same issue by using the mailboxprocessor, part of FSharp core library, that allows to process messages in a single thread.

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

The entire expression is wrapped in an async block and the processor.PostAndReply function is used to send the function f to the mailboxprocessor and wait for the result.

This approach is extremely simple and effective but may slow down the processing of commands if the aggregate is involved in a long running transaction.

In some cases we may prefer explicit locks, in some cases it could be more convenient to use the mailboxprocessor and in other cases we may not use any lock at all!

Consider for example the addTodo:  if you do it without any locking and wighout any single thread mailboxprocessor based action, then you may end up with in inconsistencies like storing an event that is actually invalid (because the todo contains an invalid tag, or the todo is duplicated).
In that case the event is stored by given that it is inconsistent, it will be ignored when the aggregate will be rebuilt.


Another example is the following, about sending commands to more aggregates.
This code removes the tag with any reference to it. It build two commands and make the repository process them at the same tim.
This code remove a tag and any reference to it.

```FSharp
    member this.removeTag id =
        ResultCE.result {
            let removeTag = TagCommand.RemoveTag id
            let removeTagRef = TodoCommand.RemoveTagRef id
            let! _ = runTwoCommands<TagsAggregate, TodosAggregate, TagEvent, TodoEvent> storage removeTag removeTagRef
            return ()
        }
```

With reference to the example involving the "under" of a command: the runTwoCommands function is executed in a transactional context if the storage supports multiple streams transactions (i.e. Postgres or in memory).
In the case the storage does not support multiple streams transactions (i.e. Eventstoredb) the runTwoCommands function will execute the two commands in a sequence and uses the undoer (if provided) to rollback the effect of the first command in case the second command fails.

Source: [App.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/App.fs)















