# Entities

With __Sharpino__ we essentially manage collections of entities.  We can think about them in the same way as we used to while approaching the "traditional" application development: records in a relational database. 
In general, any entity has no direct reference to any other entity. However it  may reference other entities them by their id (as an extenal reference, similar to the concept of foreign key).  So for instance a _todo_ may contain a list of ids of  tags  and a list of ids of categories that are related to it (any todo may references zero or more tags and zero or more categories).
This guideline is not strict: for instance, if you have an _order_ entity and an _order item_ entity in the same place then you may prefer that order items are a list of actual orderitem type rather than a list of ids. This will just make harder to move orderitems in different aggregate (but anyway, that would be really unlikely, in term of a more "classical" concept of aggregate).

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
Note that sometimes I can use the term "model" to refer to an entity or to a collection of entities. For instance, the "Todos" model is a collection of "Todo" entities.

It is a good practice to defin3 an explicit "Zero" static member for entitiy. The aggregate will use it to define its own _Zero_ static member which is mandatory for any aggregate: An aggregate expose its _Zero_ by means of the _Zero_ of its entities.
The _Zero_ static member for an aggregate is mandatory and represents the initial state of the aggregate.  We will need it to rebuild the state of the aggregate when there are no events stored for that aggregate.
Note that in some popular full-stack development FSharp technologies (["Safe stack"](https://safe-stack.github.io) and specifically [Fable remoting](https://github.com/Zaid-Ajaj/Fable.Remoting)) the definitions of the entities that must be shared among the client side (Fable to Javascript node based app) and the server side (.net core web service based app)  [Fable](https://fable.io)). So the definition of the entities need to stay in a separate project shared referenced by the the client and the server projects.
This means that we would define the "Todo", the "Tag" and the "Category" types in a shared project.

Source: [TodosModel.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/entities/Todos.fs)

