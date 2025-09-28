# Context

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


Source: [TodosModel.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample.Shared/Entities.fs)
