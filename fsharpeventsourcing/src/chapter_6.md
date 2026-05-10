# The Storage & State Layer (Async First)

The persistence mechanism in Sharpino is responsible for saving events and snapshotting aggregate states. As modern web applications demand high throughput and non-blocking I/O, Sharpino's storage layer has been comprehensively upgraded to an asynchronous-first design.

## Asynchronous Event Store

Every critical member of the Event Store—whether it is retrieving events, fetching snapshots, or committing new sequences—now has a dedicated, Task-based `Async` version. This guarantees that your domain operations do not block threads while waiting on the underlying database (such as PostgreSQL).

## StateView Enhancements

The `StateView` component manages the read-side projections of the Aggregate state. 

Recent enhancements to the `StateView` ensure that it supports multiple asynchronous views of the aggregate state. These async views fully mimic the behavior of their non-async counterparts but provide the critical addition of **Cancellation Token** support.

By utilizing Cancellation Tokens, long-running state reconstruction queries can be safely aborted if a client disconnects or a timeout occurs, significantly improving the stability and resource management of the web application.
