# Models

In Sharpino models are collections of entities.
Usually a model contains references to elements of other models only indirectly, by using their id as a reference.
However there can be exceptions when some models are closely related: for instance you may rather define a model for orders and a model for orderitems in the same place, so that orders will be able to contain direct reference to orderitems instead of their ids.

A Simple model for the todo items:

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

There is a special "Zero" static member that does not matter so much for the model, but it is important for the aggregate (see the next section). Indeed the _Zero_ static member is mandatory for any aggregate.

Note that in some popular Fsharp technologies some definitions must stay in a separate project whared beteen the client and the server separate projects.
(See Fable remoting for more informations about that).

Source: [TodosModel.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Sample/models/TodosModel.fs)

