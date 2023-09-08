# Entities

With __Sharpino__ we essentially manage collections of entities. We can think about them in the same way as we used to while approaching the "trantitional" application development. Records in a relational database. 
In general, any entity has no direct reference to any other entity but may reference them by their id (as an extenal reference).  So for instance a todo may contain a list of tag ids and a list of category ids that are related to it (any todo may have zero or more tags and zero or more categories).
However there can be exceptions for models that are closely related: for instance, you may rather define an _order_ entity and an _order item_ entity in the same place so that any orders can contain a direct reference to order items instead of the id of order items (by using this approach you would represent the order items as a list of order items instead of a list of order item ids).

Here there is my todo entity definition:

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
Sometimes I will use the term "model" to refer to an entity or to a collection of entities. For instance, the "Todos" model is a collection of "Todo" entities.

You've seen a special "Zero" static member. The aggregate will use it to define its own _Zero_ static member which is mandatory for any aggregate.
The _Zero_ static member is the initial state of the aggregate and it is used to rebuild the state of the aggregate when there are no events stored for that aggregate.
Note that in some popular full-stack development FSharp technologies (["Safe stack"](https://safe-stack.github.io) and specifically [Fable remoting](https://github.com/Zaid-Ajaj/Fable.Remoting)) definitions of entities that must be accessed on the client side (using Fsharp technology transpiled in javascript, i.e. [Fable](https://fable.io)) must stay in a separate project shared between the client and the server. 
In that case, we would define the "Todo", the "Tag" and the "Category" types in a shared project so an eventual Fable/Elmish client application can use them.

Source: [TodosModel.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Sample/models/TodosModel.fs)

