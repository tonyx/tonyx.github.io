# Entities

I am representing in this example the Todo entity and a Todo context

This is the Todo entity definition:

```FSharp
    type Todo =
        {
            Id: Guid
            CategoryIds : List<Guid>
            TagIds: List<Guid>
            Description: string
        }
    type Todos =
        {
            todos: List<Todo>
        }
        with
            static member Zero =
                {
                    todos = []
                }
```

The Todos context needs also some other members (to handle the stream name, the serialize/deserialize functions, and the snapshots interval...)

The Context will define a _Zero_ static member (initial state).
In case we use 
 [Fable remoting](https://github.com/Zaid-Ajaj/Fable.Remoting))
then we need to share the definition of the entities between the client and the server side.
(Shared project)

Source: [TodosModel.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample.Shared/Entities.fs)
