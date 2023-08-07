# Aggregates

An aggregate is an instance of a class (or more properly, a record) that handles one or more models. We will be able to recreate the state of the aggregate by processing the stored events by an essential function called _evolve_. Such   _events_ are closely related to some of the members of the aggregate. 

Given that the state of the aggregate is a function of the related events processed and stored, an aggregate needs the following information associated that I defined as mandatory, static members:

- __Zero__: the instance of the aggregate in its initial state. 
It is needed to be able to provide the state of the aggregate when there is no event stored related to that aggregate.
- __StorageName__ and  __Version__: this combination uniquely identifies the aggregate and lets the storage know in which stream to store events and snapshots.

- - __LockObject__: (_warning: the lockobject concept is obsolete. It was meant to handle single-thread processing but at the moment I am providing an actor model based mailboxprocessor to ensure single-thread chain command->events->eventstoring, so you will skip this part_). Before introducing the _mailboxprocessor_ (i.e. an actor model based single thread message processor), the repository was supposed to use aggregate locks while processing commands and storing related events ensuring consistency. An application layer was also supposed to use them explicitly to ensure inter-aggregate integrity (invariant conditions involving models handled by separate aggregates).
- __SnapshotsInterval__: the number of the events that can be stored after a snapshot before creating a new snapshot (i.e. the number of events between snapshots)

Note that in the _in memory_ or _Postgres_ storage implementation the state of any aggregate is rebuilt starting from the last available snapshot and applying the events that are after the snapshot.
At the moment I have no specific strategy for handling snapshots and intervals between snapshots for the EventStoreDb. 

Example:
```FSharp
    type TodosAggregate =
        {
            todos: Todos
            categories: Categories
        }
        static member Zero =
            {
                todos = Todos.Zero
                categories = Categories.Zero
            }
        static member StorageName =
            "_todo"
        static member Version =
            "_01"
        static member SnapshotsInterval =
            15
```

Some members of the aggregate virtually change the state of the aggregate in the sense that they return an instance of the aggregate in a different state or an error. It actually means that such members will _add/update/delete_  some of their models (still in a functional/immutable way i.e. returning the model with changes). Those members will be associated with aggregate _events_ (see next section).
In the following example, the aggregate of the Todos manages also the categories model and so it can check the validity of the categories referenced by a todo before adding it (to preserve the rule that you can add only todo with category-Ids related to existing categories).
It uses the "result" computational expression included in the FsToolkit.ErrorHandling library supports the possibility to handle errors in a _railway-oriented programming_ style.

Example:
```FSharp
    member this.AddTodo (t: Todo) =
        let checkCategoryExists (c: Guid ) =
            this.categories.GetCategories() 
            |> List.exists (fun x -> x.Id = c) 
            |> boolToResult (sprintf "A category with id '%A' does not exist" c)

        result
            {
                let! categoriesMustExist = t.CategoryIds |> catchErrors checkCategoryExists
                let! todos = this.todos.AddTodo t
                return 
                    {
                        this with
                            todos = todos
                    }
            }
```
Recap: we can protect invariant conditions related to models that are part of the same aggregate without the need for any explicit transaction because at that level there is no awareness of the storage. Still, we can protect other invariant conditions that may involve models residing in different aggregates anyway in a different way:  at the service application layer (see in the next sections).

Source: [TodosAggregate.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/aggregates/Todos/Aggregate.fs)
