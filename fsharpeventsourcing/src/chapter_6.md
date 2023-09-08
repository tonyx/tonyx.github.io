# Repository
The repository has the responsibility of:
- getting the state of an aggregate
- trying to run commands passed, and eventually storing the related events returned by the command.
- making periodic snapshots, according to the SnapshotsInterval parameter of the aggregate.
(Remember that snapshots are explicitly used only in the Postgres and in-memory storage implementations)

Here is an example of the private member that retrieves the last snapshot:

```FSharp
    let inline private getLastSnapshot<'A 
        when 'A: (static member Zero: 'A) 
        and 'A: (static member StorageName: string)
        and 'A: (static member Version: string)>
        (storage: IStorage) = 

        ResultCE.result {
            let! result =
                match storage.TryGetLastSnapshot 'A.Version 'A.StorageName  with
                | Some (snapId, eventId, json) ->
                    let state = SnapCache<'A>.Instance.Memoize (fun () -> json |> deserialize<'A>) snapId
                    match state with
                    | Error e -> Error e
                    | _ -> (eventId, state |> Result.get) |> Ok
                | None -> (0, 'A.Zero) |> Ok
            return result
        }
```
This function uses the storage to retrieve a triple of the snapshot-Id, the related eventId and the snapshot itself, serialized as json.
Note that the snapshot may be cached in memory so that the deserialization is done only once.

To get the current state of an aggregate we need to get the last snapshot and the events that are after the snapshot.

Here is an older version of how to get the state:

```Fsharp
    let inline getState<'A, 'E
        when 'A: (static member Zero: 'A)
        and 'A: (static member StorageName: string)
        and 'A: (static member Version: string)
        and 'A: (static member LockObj: obj)
        and 'E :> Event<'A>>(storage: IStorage) = 

        let snapIdStateAndEvents()  =
            lock 'A.LockObj ( fun _ ->
                ResultCE.result {
                    let! (id, state) = getLastSnapshot<'A> storage
                    let events = storage.GetEventsAfterId 'A.Version id 'A.StorageName
                    let result =
                        (id, state, events)
                    return result
                }
            )

        ResultCE.result {
            let! (lastSnapshotId, state, events) = snapIdStateAndEvents()
            let lastEventId =
                match events.Length with
                | x when x > 0 -> events |> List.last |> fst
                | _ -> lastSnapshotId 
            let! events' =
                events |>> snd |> catchErrors deserialize<'E>
            let! result =
                events' |> evolve<'A, 'E> state

            return (lastEventId, result)
        }
```

Now I show the current implementation that enables aggregate-state caching:

```FSharp
    let inline getState<'A, 'E
        when 'A: (static member Zero: 'A)
        and 'A: (static member StorageName: string)
        and 'A: (static member Version: string)
        and 'E :> Event<'A>>(storage: IStorage) = 

        let snapIdStateAndEvents()  =
            async {
                return
                    ResultCE.result {
                        let! (id, state) = getLastSnapshot<'A> storage
                        let events = storage.GetEventsAfterId 'A.Version id 'A.StorageName
                        let result =
                            (id, state, events)
                        return result
                    }
            }
            |> Async.RunSynchronously

        let eventuallyFromCache = 
            fun () ->
                ResultCE.result {
                    let! (lastSnapshotId, state, events) = snapIdStateAndEvents()
                    let lastEventId =
                        match events.Length with
                        | x when x > 0 -> events |> List.last |> fst
                        | _ -> lastSnapshotId 
                    let! events' =
                        events |>> snd |> catchErrors deserialize<'E>
                    let! result =
                        events' |> evolve<'A, 'E> state
                    return (lastEventId, result)
                }
        let lastEventId = 
            async {
                return storage.TryGetLastEventId 'A.Version 'A.StorageName |> Option.defaultValue 0
            } 
            |> Async.RunSynchronously
        StateCache<'A>.Instance.Memoize (fun () -> eventuallyFromCache()) (lastEventId, 'A.StorageName)
```

In the above code, the state is a function of eventId, and so we can use this eventId as the key of a cache that stores the state of the aggregate.

Note that here the evolve function is used, which is part of the core library.

There are actually two similar evolve implementations:

The basic implementation of the evolve is the one that cannot forgive any inconsistency in the  events passed as parameters with the current aggregate state:

```Fsharp
    let inline evolveUNforgivingErrors<'A, 'E when 'E :> Event<'A>> (h: 'A) (events: List<'E>) =
        events
        |> List.fold
            (fun (acc: Result<'A, string>) (e: 'E) ->
                match acc with
                    | Error err -> Error err 
                    | Ok h -> h |> e.Process
            ) (h |> Ok)
```


Here is an implementation of the evolve that skips eventual inconsistent events:

```Fsharp
    let inline evolve<'A, 'E when 'E :> Event<'A>> (h: 'A) (events: List<'E>): Result<'A, string> =
        let rec evolveSkippingErrors (acc: Result<'A, string>) (events: List<'E>) (guard: 'A) =
            match acc, events with
            | Error err, _::es -> 
                // you may want to print or log this
                printf "warning: %A\n" err
                evolveSkippingErrors (guard |> Ok) es guard
            | Error err, [] -> 
                // you may want to print or log this
                printf "warning: %A\n" err
                guard |> Ok
            | Ok state, e::es ->
                let newGuard = state |> e.Process
                match newGuard with
                | Error err -> 
                    // use your favorite logging library here
                    printf "warning: %A\n" err
                    evolveSkippingErrors (guard |> Ok) es guard
                | Ok h' ->
                    evolveSkippingErrors (h' |> Ok) es h'
            | Ok h, [] -> h |> Ok

        evolveSkippingErrors (h |> Ok) events h
```

Code in [Repository.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Lib/Repository.fs) and
[Core.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Lib/Core.fs)


There is also an experimental repository based on a publish/subscribe storage model (Eventstoredb).
See _lightrepository_

 [LightRepository.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Lib/LightRepository.fs) 

