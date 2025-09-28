# Refactoring Aggregates

If you need to change an aggregate (or context) for new requirements, you need to refactor the aggregate so that it can handle the old and the new formats.

The technique is based on being able to create an aggregate that mimics the old one and an upcast function from the old one to the new one.

It is convenient to do a bulk upcast of all the existing aggregates by making a snapshot of all of them.
The new snapshot will use the new aggregate format, so the definition of the old aggregate will be unnecessary after the bulk upcast and resnapshot.

To do an upcast, you can create a shadow aggregate that mimics the old one in the same module of the current one, provided that the module is defined as rec (recursive).

The mechanism is based on handling the failure of the Deserialization by using as fallback the Deserialization using the old aggregate and then the upcast function to the new one.

Note: it may depend on the serialization library and its configuration as it is not always true that you can deserialize the old aggregate changing its target type with a different name than the original one. 

The provided examples are all using FSPickler configured in a way that it allows the upcast from the old aggregate with a different name to the new one.
