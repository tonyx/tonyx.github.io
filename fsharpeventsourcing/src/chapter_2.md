# Aggregates

An aggregate is an instance of a class (or more properly, a record) that handles one or more collections of entities. We will be able to recreate the state of the aggregate by processing the stored events by the  _evolve_ function. As we will see later, the _events_ are closely related to some of the members of the aggregate. 

Given that the state of the aggregate is a function of the related events processed and stored, an aggregate needs the following information associated that I defined as mandatory, static members:

- __Zero__: the instance of the aggregate in its initial state. 
It is needed to be able to provide the state of the aggregate when there is no event stored related to that aggregate.
- __StorageName__ and  __Version__: this combination uniquely identifies the aggregate and lets the storage know in which stream to store events and snapshots.

- - __LockObject__: (_warning: the lockobject concept is obsolete. It was meant to handle single-thread processing but at the moment I am providing an actor model based mailboxprocessor to ensure single-thread chain command->events->eventstoring, so you will skip this part_). Before introducing the _mailboxprocessor_ (i.e. an actor model single thread message processor), the repository was supposed to use aggregate locks while processing commands and storing related events ensuring consistency using pessimistic locking. An application layer was also supposed to use them explicitly to ensure inter-aggregate integrity (invariant conditions involving models handled by separate aggregates).
- __SnapshotsInterval__: the number of the events that can be stored after a snapshot before creating a new snapshot (i.e. the number of events between snapshots)

In all my current implementations of storage  (_in memory_,  _Postgres_ and _EventstoreDb_)  the state of any aggregate is rebuilt starting from the latest available snapshot and applying the events that are after the snapshot. The current state of any aggregate may also be cached.


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

Some members of the aggregate virtually change the state of the aggregate in the sense that they return an instance of the aggregate in a different state or an error. It means that such members will _add/update/delete_  some of their entities (still in a functional/immutable way i.e. returning a new collection with some changes). Those members will be associated with aggregate _events_ (see next section).
In the following example, the aggregate of the Todos manages also the categories, so it will be able to check the validity of the categories referenced by a todo before adding it (to preserve the rule that you can add only todo with category-Ids related to existing categories).
It uses the "result" computational expression included in the FsToolkit.ErrorHandling library which supports the [_railway-oriented programming_](https://fsharpforfunandprofit.com/rop/) pattern of handling errors.

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
Wrap-up: We can protect invariant conditions related to models that are part of the same aggregate without the need for any explicit transaction because at that level there is no awareness of the storage. Still, I will show that we can protect other invariant conditions that may involve models residing in different aggregates at the service application layer (see in the next sections) running commands involving multiple streams. This sometimes will involve the use of explicit transactions when supported by the storage (i.e. Postgres), and sometimes will involve the "undoer": a special command attached to a command to undo the changes made by the command itself (see the next sections).

Source code: [TodosAggregate.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/aggregates/Todos/Aggregate.fs)
