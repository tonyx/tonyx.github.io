# Commands

(this section needs an update as we have a new )

A Command type can be represented by a Discriminated Union. Executing the command on a specific context or aggregate means returning a new state and, accordingly, a list of events, or an error.
You can also specify _"command undoers"_, that allow you to compensate the effect of a command. An undoer returns a new function that in the future can be executed to return the events that can reverse the effect of the command itself.
For example, the "under" of AddTodo is the related RemoveTodo (see next paragraph).

In the following code we can see the signature for any state viewers for any context or aggregate. State viewer corresponds to read models: they will provide the current state of aggregate or context. Typically, that state may come from a cache, from the event store (by processing the events) or from a topic of Kafa (or eventually any other message/event broker, even though I haven't implemented completed any of them yet).

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

Example:

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


Any command must ensure that it will return Result.OK (and therefore, one or more events) only if the events to be returned, when processed on the current state, return an Ok result, i.e. a valid state (and no error). 

There are two version of the _evolve_: one tolerates inconsistent events and another one will fail in case just an event will return an error.
The way the evens are stored in the event store ensures that no stored event will return inconsistent state when processed. Therefore future releases of the library will probably use by default the unforgiving version of the _evolve_ function.
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





This is the abstract definition of an of the undoer of an aggregate.
```FSharp
    type AggregateCommandUndoer<'A, 'E> = Option<'A -> AggregateViewer<'A> -> Result<unit -> Result<'A * List<'E>, string>, string>>

```

The meaning is:
Extract from the current state of the aggregate useful info for a future "rollback"/"undo" and return a function that, when applied to the current state of the aggregate, will return the events that will "undo" the effect of this command.

You may need undoers if the eventstore doesn't support multiple stream transactions or if you will use a distributed architecture with many nodes handling different streams of events.
By using PostgresSQL as event store you can just set the undoer to None as the event store will handle the cross-streams transactions for us. 

[Commands.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/Domain/Tags/Commands.fs)


