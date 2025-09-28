# Application service layer

An application service layer implements the logic and makes business logic calls, or queries, available.
Here is one of the simplest examples of an entry for a service involving a single context, by building and running an AddTag command.

```FSharp
        member this.AddTag tag =
            result {
                let! result =
                    tag
                    |> AddTag
                return result 
            }
```

The service layer sends commands to the Command Handler so that this one can run it producing and storing the related events, returning the EventStore Ids of the stored events.

## Running two commands to different clusters

This code removes the tag with any reference to it. It builds two commands and makes the repository process them at the same time.
This code removes a tag and any reference to it.

```FSharp
    member this.removeTag id =
        ResultCE.result {
            let removeTag = TagCommand.RemoveTag id
            let removeTagRef = TodoCommand.RemoveTagRef id
            return! runTwoCommands<Tag, Todo, TagEvent, TodoEvent> eventStore eventBroker removeTag removeTagRef
        }
```

Source: [App.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/App.fs)
















