# Rabbitmq

For each aggregate we can define a "consumer" that is able to listen to the events of that state

Some tests are instrumented to be able to test this configuration by adding some artificial delays in those tests to allow the propagation of the messages. Those examples gets the state of the involved aggregates by those consumers instead of accessing the event store. RabbitMq must be running on localhost. (in my configuration it's just about running rabbitmq-server. It must of course be ok running it on docker).
