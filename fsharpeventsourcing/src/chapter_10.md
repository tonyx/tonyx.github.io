# Apache Kafka
We can use Apache Kafka to notify events after storing them.
The style is the outbox pattern.
Examples of usage are [here](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/AppVersions.fs).
Some tests are instrumented to eventually listen to Apache Kafka, see [here](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample.Test/MultiVersionsTests.fs).
Any "view" node may build its own view of the cluster state by listening to the events.