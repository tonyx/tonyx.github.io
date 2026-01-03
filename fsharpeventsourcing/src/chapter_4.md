# Commands

A _command_ (or _aggregate command_) is an object that when executed against a specific context or aggregate will produce a new state and, accordingly, a list of events (or an error).

It provides also an optional _undoer_ that can be used to compensate the effect of the command itself if needed (perhaps in case of distributed transactions or if the event store doesn't support multiple stream transactions).

```FSharp
    type AggregateCommand<'A, 'E when 'E :> Event<'A>> =
        abstract member Execute: 'A -> Result<'A * List<'E>, string>
        abstract member Undoer: AggregateCommandUndoer<'A, 'E>
```

The AggregateCommandUndoer is defined as follows:
```FSharp
    type AggregateCommandUndoer<'A, 'E> = Option<'A -> AggregateViewer<'A> -> Result<unit -> Result<'A * List<'E>, string>, string>>
```

It is optional (can be None) and when defined it provides a function that, when applied to the current state of the aggregate and to an aggregate viewer (to get the current state of the aggregate in the future), will return a function that when executed will return a result with the list of events that can "undo" the effect of the command itself.

The _undoer_ depends uses the _AggregateViewer_ defined as follows:
```FSharp
    type AggregateViewer<'A> = AggregateId -> Result<EventId * 'A,string>
```

 Executing the command on a specific context or aggregate instance means returning a new state and, accordingly, a list of events, or an error.

In the following code we can see the signature for any state viewers for any context or aggregate. State viewer corresponds to read models: they will provide the current state of aggregate or context. Typically, that state may come from a cache, from the event store (by processing the events) or from a topic of Kafa (or eventually any other message/event broker, even though I haven't implemented completed any of them yet).

Here is the complete definition of commands and related types from the _core_ module:

```FSharp

    type StateViewer<'A> = unit -> Result<EventId * 'A, string>
    type AggregateViewer<'A> = AggregateId -> Result<EventId * 'A,string>
   
   
    type Aggregate<'F> =
        abstract member Id: AggregateId 
        abstract member Serialize: 'F
    
    type Event<'A> =
        abstract member Process: 'A -> Result<'A, string>

    type CommandUndoer<'A, 'E> =  Option<'A -> StateViewer<'A> -> Result<unit -> Result<List<'E>, string>, string>>
    type AggregateCommandUndoer<'A, 'E> = Option<'A -> AggregateViewer<'A> -> Result<unit -> Result<'A * List<'E>, string>, string>>
    type Command<'A, 'E when 'E :> Event<'A>> =
        abstract member Execute: 'A -> Result<'A * List<'E>, string>
        abstract member Undoer: CommandUndoer<'A, 'E>
        
    type AggregateCommand<'A, 'E when 'E :> Event<'A>> =
        abstract member Execute: 'A -> Result<'A * List<'E>, string>
        abstract member Undoer: AggregateCommandUndoer<'A, 'E>
```

Now a complete example related to one of the examples provided.

```FSharp

    type CourseEvents =
        | StudentEnrolled of StudentId
        | StudentUnenrolled of StudentId
        | Renamed of string
        interface Event<Course> with
            member this.Process (course: Course) =
                match this with
                | StudentEnrolled studentId -> course.EnrollStudent studentId
                | StudentUnenrolled studentId -> course.UnenrollStudent studentId
                | Renamed name -> course.Rename name
       
        static member Deserialize (x: string): Result<CourseEvents, string> =
            try
                JsonSerializer.Deserialize<CourseEvents> (x, jsonOptions) |> Ok
            with
            | ex ->
                Error (ex.Message)
        member this.Serialize =
            JsonSerializer.Serialize (this, jsonOptions)
```


A command returns returns a new state and a list of on e or more events:

```fsharp
    type CourseCommands =
        | EnrollStudent of StudentId
        | UnenrollStudent of StudentId
        | Rename of string
        interface AggregateCommand<Course, CourseEvents> with
            member this.Execute (course: Course) =
                match this with
                | EnrollStudent studentId ->
                    course.EnrollStudent studentId
                    |> Result.map (fun s -> (s, [ StudentEnrolled studentId]))
                | UnenrollStudent studentId ->
                    course.UnenrollStudent studentId
                    |> Result.map (fun s -> (s, [ StudentUnenrolled studentId]))
                | Rename name ->
                    course.Rename name
                    |> Result.map (fun s -> (s, [ Renamed name]))    
            member this.Undoer = None
```

In this example the command implements the Execute by relying on the _EnrollStudent_ member as a "decide" function. So if _EnrollStudent_ returns an Ok result with the new state, then the command will return also the related event to be stored.

Once the events are produced by the command, they can be stored in the event store and then used to evolve the state of the aggregate or context.

There are two version of the _evolve_: one tolerates inconsistent events and another one will fail in case just an event will return an error.

Currently the policy is using the "forgiving" verion only as a fallback, by logging the error and skipping the inconsistent events.  

## Undoer

An undoer would be useful in case the only way to make two streams consistent is to "rollback" the effect of a command on one stream if the command on another stream fails.

(this should be handled at the application level, though)

Unfortunately there is no  simple way to express the undoer, yet. The good news is that an undoer is optional, and generally is set to None.

```fsharp
    member this.Undoer =
        match this with
        | EnrollStudent studentId ->
            (
                fun (course: Course) (viewer: AggregateViewer<Course>) ->
                    result {
                        return 
                            fun () ->
                                result {
                                    let! _, state = viewer course.Id.Id
                                    let result =
                                        state.UnenrollStudent studentId
                                        |> Result.map (fun s -> s, [ StudentUnenrolled studentId])
                                    return! result
                                }
                    }    
            )
            |> Some
        | UnenrollStudent studentId ->
            (
                fun (course: Course) (viewer: AggregateViewer<Course>) ->
                    result {
                        return 
                            fun () ->
                                result {
                                    let! _, state = viewer course.Id.Id
                                    let result =
                                        state.EnrollStudent studentId 
                                        |> Result.map (fun s -> s, [StudentEnrolled studentId])
                                    return! result
                                }
                    } 
            )
            |> Some
        | Rename _ ->
            None

```

The meaning is:
if a command is already executed and the related events are stored, we have a way to ask to that command to provide a function, that can be executed in the future to return another function that when executed will return a result with the list of events that can "undo" the effect of the command itself.

By using PostgresSQL as event store you can just set the undoer to None as the event store will handle the cross-streams transactions for us. 

[Commands.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/Domain/Tags/Commands.fs)


