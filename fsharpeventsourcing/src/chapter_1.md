# Models

In Sharpino models are collections of entities.
Usually a model contains references to elements of other models only indirectly, for example by including their ids.
However there can be exceptions when the model are closely related: for instance you may find convenient to define a model for orders and define a model for orderitems at the same place, so that orders can contain direct reference to orderitems instead of referencing them by their ids.

A Simple model for todo items:

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

As I will hopefully show in future documents, in an application using technologies where entities definition must be shared between client and server, like inf Fable Remoting, the definition of the single entities of models must stai in a shared project, available to the client (Fable/Elmish) and to the server. So the "Todo type" will need to be in a shared project.

Source: [TodosModel.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Sample/models/TodosModel.fs)

