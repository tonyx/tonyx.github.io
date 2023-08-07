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

As I can show in future documents, in an application using the Fable Remoting technology, the definition of the single entities of models must be shared with the Fable Client in a shared project, and that means that in our case the "type Todo" needs being in a Shared project. (shared between a Fable/Elmish client project).

Source: [TodosModel.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Sample/models/TodosModel.fs)

