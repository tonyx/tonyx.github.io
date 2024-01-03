# Entities

I can represent entities as records. They need a unique identifier (ID). 
I can reference other entities by their Ids.

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

The Context will define a _Zero_ static member (initial state).
In case we use 
 [Fable remoting](https://github.com/Zaid-Ajaj/Fable.Remoting))
then we need to share the definition of the entities between the client and the server side.
(Shared project)

Source: [TodosModel.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample.Shared/Entities.fs)
