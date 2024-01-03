# Application service layer

An application service layer implements multiple context logic.
Here is one of the simplest examples of an entry for a service involving a single context, by building and running an AddTag command.

```FSharp
        member this.AddTag tag =
            result {
                let! result =
                    tag
                    |> AddTag
                    |> runCommand<TagsContext, TagEvent> storage eventBroker tagsStateViewer
                return result 
            }
```

As in the previous example, the service layer sends commands to the Command Handler so that this one can run it producing and storing the related events, returning the EventStore Ids of the stored events and the KafkaDeliveryResult (if Kafka broker is enabled).

We may adopt strong and immediate consistency by enforcing the use of a pessimistic lock on the command handler.
An example of explicit use of lock:

```FSharp
    member this.addTodo todo =
        lock TagsCluster.LockObj <| fun () ->
            result {
                let! (_, tagState) = getState<TagsCluster, TagEvent>(storage)
                let tagIds = tagState.GetTags() |>> (fun x -> x.Id)

                let! tagIdIsValid =    
                    (todo.TagIds.IsEmpty ||
                    todo.TagIds |> List.forall (fun x -> (tagIds |> List.contains x)))
                    |> boolToResult "A tag reference contained is in the todo is related to a tag that does not exist"

                return! 
                    todo
                    |> TodoCommand.AddTodo
                    |> (runCommand<TodosCluster, TodoEvent> storage)
            }
```

The todo can be added only if it contains valid tag references.

## Running two commands to different clusters

This code removes the tag with any reference to it. It builds two commands and makes the repository process them at the same time.
This code removes a tag and any reference to it.

```FSharp
    member this.removeTag id =
        ResultCE.result {
            let removeTag = TagCommand.RemoveTag id
            let removeTagRef = TodoCommand.RemoveTagRef id
            return! runTwoCommands<TagsCluster, TodosCluster, TagEvent, TodoEvent> storage removeTag removeTagRef
        }
```

The _runTwoCommands_ uses the undoer if the current event store requires it (i.e. it lacks multiple stream transactions).

Source: [App.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/App.fs)
















