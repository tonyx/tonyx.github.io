# Models

In Sharpino models are collections of entities.
Usually a model contains references to elements of other models only indirectly, for example by including their ids.
However there can be exceptions when some models are closely related: for instance you may find convenient to define a model for orders and a model for orderitems in the same place, so that orders will be able to contain direct reference to orderitems instead of their ids.

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

Note that in an application using technologies where entities definition must be shared between client and server, like Fable Remoting, the definition of the single entities of models must stay in a shared project, available to the client (Fable/Elmish) and to the server as well. So the "Todo type" definition will need to be in a shared project.

Source: [TodosModel.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Sample/models/TodosModel.fs)

