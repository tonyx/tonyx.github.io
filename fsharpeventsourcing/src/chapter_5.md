# Application service layer

An application service layer provides services that use Repository and Storage to get the state and/or send commands to one or more clusters (eventually in an atomic/transactional way with potential performance issues) and store the related events.

Here is one of the simplest examples of an entry for a service involving a single cluster, by building and running an AddTag command.


```FSharp
    member this.addTag tag =
        result {
            let! _ =
                tag
                |> AddTag
                |> (runCommand<TagsCluster, TagEvent> storage)
            return ()
        }
```

The service layer sends commands to the Command Handler so that this one can run it producing and storing the related events.

Here is an example of using an explicit lock for consistency in an operation involving two clusters (we check that the tag is valid before adding the todo and inhibit write access to tags while the todo is being added).

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

I may just have the same effect by passing the entire block to a mailboxprocessor:

```FSharp
        member this.AddTodo todo =
            let f = fun () ->
                ResultCE.result {
                    let! (_, tagState) = storage |> getState<TagsCluster, TagEvent> 
                    let tagIds = tagState.GetTags() |>> (fun x -> x.Id)

                    let! tagIdIsValid =    
                        (todo.TagIds.IsEmpty ||
                        todo.TagIds |> List.forall (fun x -> (tagIds |> List.contains x)))
                        |> boolToResult "A tag reference contained in the todo is related to a tag that does not exist"

                    let! _ =
                        todo
                        |> TodoCommand.AddTodo
                        |> runCommand<TodosCluster, TodoEvent> storage
                    let _ = 
                        storage
                        |> mkSnapshotIfInterval<TodosCluster, TodoEvent>
                return ()
            }
            async {
                return processor.PostAndReply (fun rc -> f, rc)
            }
            |> Async.RunSynchronously
```

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

The _runTwoCommands_ uses the undoer of the storage requires it.

Source: [App.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/App.fs)















