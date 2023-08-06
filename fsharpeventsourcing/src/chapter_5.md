# Application service layer

An application service layer provides services that actually uses  Respository and Storage to send commands and get the state of the aggregates.

Here is an example of an entry for the services involving a single aggregate.


```FSharp
    member this.addTag tag =
        ResultCE.result {
            let! _ =
                tag
                |> AddTag
                |> (runCommand<TagsAggregate, TagEvent> storage)
            return ()
        }
```

The service layer send commands to the repository so that it can produce and store related events.

The following example shows a service function that uses two aggregates and an explicit lock.

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















