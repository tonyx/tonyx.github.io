# Models

In __Sharpino__ the models are collections of entities.
The general case is that a model contains no direct reference to any other model but may reference them by their id. So for instance a todo may contain a list of tag ids and a list of category ids.
This is the general case because it is the most flexible one: you can change the model without any need to change the other models.
However there can be exceptions for models that are closely related: for instance, you may rather define a model for orders and a model for order items in the same place so that any orders can contain a direct reference to order items instead of the id of order items.

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

You've seen a special "Zero" static member. The aggregate will use it to define its own _Zero_ static member which is mandatory for any aggregate.
The _Zero_ static member is the initial state of the aggregate and it is used to rebuild the state of the aggregate when there are no events stored for that aggregate.
Note that in some popular full-stack development FSharp technologies some definitions must stay in a separate project shared between the client and the server separate projects.
In that case, we would define the "Todo", the "Tag" and the "Category" types in a shared project.
(See Fable remoting for more information about that).

Source: [TodosModel.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Sample/models/TodosModel.fs)

