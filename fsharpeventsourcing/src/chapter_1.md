# Models

In Sharpino models are collections of entities.
Usually a model contains references to elements of other models only indirectly, by using their unique identifiers (id) as a reference.
However there can be exceptions for models that are are closely related: for instance you may rather define a model for orders and a model for orderitems in the same place, so that orders will be able to contain direct reference to orderitems instead of their ids.

Here there is a simple model of the todo items:

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

You've seen a special "Zero" static member. It is important, and also mandatory, for an aggregate to define a _Zero_ static member and so it will use the zero member of any model as well.

Note that in some popular full stack development FSharp technologies some definitions must stay in a separate project shared beteen the client and the server separate projects.
(See Fable remoting for more informations about that).

Source: [TodosModel.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Sample/models/TodosModel.fs)

