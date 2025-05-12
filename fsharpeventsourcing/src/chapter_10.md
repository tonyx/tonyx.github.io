# Apache Kafka

We will see later if and when an event broker is needed.

Anyway the mechanism should be: after the event is stored in the event store, it will be published to a queue/topic/message bus.

The command handler will still use the event store to get the state of the context and to store the events.

An application layer may read the state of some aggregate/context by listening the events.

