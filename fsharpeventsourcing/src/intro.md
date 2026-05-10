# Introduction to Sharpino and Event Sourcing

[Sharpino](https://github.com/tonyx/Sharpino) is a lightweight, modern F# event sourcing framework designed to power robust, predictable, and highly scalable backends.

## The Paradigm Shift

Traditional CRUD applications rely on structural state stored in a relational database, constantly overwritten as the system changes. Event Sourcing represents a paradigm shift: moving from tracking the *current structural state* to tracking the *behavioral domain events* that caused the state to change. 

In Sharpino, the immutable sequence of events is the single source of truth. The current state is simply a projection or "fold" of those events over time. This approach guarantees an absolute audit trail and enables powerful temporal queries.

## The Canonical Example: BlazorBookLibrary

While Sharpino has numerous examples in its primary repository, the best way to understand its power in a real-world setting is by exploring the canonical, production-ready example: [blazorBookLibrary](https://github.com/tonyx/blazorBookLibrary). 

This project demonstrates all the advanced features discussed throughout this book:
- Using Pure Aggregates with specific Value Object IDs.
- Eliminating bidirectional relationships in favor of unidirectional flow and Materialized Detail Views.
- High-performance, distributed L1/L2 caching using SQL and backplanes.
- A fully asynchronous architecture capable of robust web-scale workloads.
- Seamless integration between an F# domain backend and a C# Blazor frontend.

This book will guide you through the principles of building event-sourced systems using Sharpino, leveraging the architectural blueprints provided by `blazorBookLibrary`.