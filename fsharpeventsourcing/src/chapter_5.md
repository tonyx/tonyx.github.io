# Application service layer

An application service layer provides services by bulding and sending commands related to one ore more aggregates. 

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

According to a business rule, the todo can be added only if it contains valid tag references.
The fact that the block seen is protected by the lock ensures that this rule is respected: according to the mentioned business rule, I want to make impossible that an event related to the tags is stored between the moment the code gets the tagId list and the moment the code adds the todo.
That would have the effect of making the todo invalid.

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

Source: [App.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Sample/App.fs)















