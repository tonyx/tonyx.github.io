# Repository
The repository has the responsibility of:
- getting the state of an aggregate
- trying to run commands passed, and eventually storing the related events.
- making periodic snapshots, according to the SnapshotsInterval parameter of the aggregate.

Here an example of the private member that retrieve the last snapshot:

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
This function use the storage to retrieve a triple of the snapshotId, the related eventId and the snapshot itself, serialized as json.

To get the current state of an aggregate we need to get the last snapshot and the events that are after the snapshot:

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

Note that here the evolve function is used, which is part of the core library and is defined as follows:

```Fsharp
    let inline evolve<'A, 'E when 'E :> Event<'A>> (h: 'A) (events: List<'E>) =
        events
        |> List.fold
            (fun (acc: Result<'A, string>) (e: 'E) ->
                match acc with
                    | Error err -> Error err 
                    | Ok h -> h |> e.Process
            ) (h |> Ok)
```

Code in [Repository.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Lib/Repository.fs) and
[Core.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Lib/Core.fs)


There is also an experimental repository based on a publish/subscribe storage model (Eventstoredb).
See lightrepository

 [LightRepository.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Lib/LightRepository.fs) 

