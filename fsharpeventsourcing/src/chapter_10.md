# Apache Kafka
We can use Apache Kafka to notify events after storing them.
The style is the outbox pattern (without db).
Examples of usage are [here](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/AppVersions.fs).
Some tests are instrumented to eventually listen to Apache Kafka, see [here](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample.Test/MultiVersionsTests.fs).
Any "view" node may build its own view of the cluster state by listening to the events.
KafkaStateViewer and KafkaAggregateStateViewer are able to build the current state by subscribing events on specific topics and specific partitions of topics.  

Command handler is able to use read models given by KafkaStateViewer.
Kafka based state viewer will be able to detect anomaly by checking the progressive id of any event.
In case of such anomaly the state viewer will be able to access to the event store to build the state.

This part is still under development to be optmized.
