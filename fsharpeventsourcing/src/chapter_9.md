# Refactoring and Testing

Event-sourced systems possess unique characteristics that require specialized approaches to both testing and code refactoring.

## Refactoring Aggregates and Events

Because the sequence of events is immutable, changing the shape of an event or the structure of an aggregate requires careful versioning. In Sharpino, you can utilize the built-in snapshotting mechanisms to baseline your aggregates, making it easier to transition between structural versions of your data.

When breaking changes to the event schema are strictly required, you can create upcasters or utilize Sharpino's event replacement features (similar to those used for GDPR ghosting) to map old event versions to new structures seamlessly during replay.

## Testing the Domain

Sharpino's pure functional approach makes testing Aggregates exceptionally straightforward. You do not need complex mock frameworks or database setups to test core business logic.

- **Testing Commands**: You simply initialize a starting State, pass it to a Command function along with test parameters, and assert that the returned `Result` contains the exact Expected Events.
- **Testing Events**: You take a starting State, apply an Event, and assert that the resulting State matches your expectations.
- **Testing Details**: You can emit a sequence of events across multiple mock aggregates and verify that the Materialized Views correctly reconstruct the composite models.

By keeping the domain logic pure and free of I/O, the majority of your test suite can run at memory speed, ensuring your application remains highly reliable as it scales.
