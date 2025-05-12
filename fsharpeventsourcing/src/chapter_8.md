# Refactoring Aggregates

If you need to change an aggregate (or context) for new requirements we need to refactor the aggregate.

The technique is based on being able to create an aggregate that mimics the old one and an upcast function from the old one to the new one.

Beside this, you may do a bulk upcast of all the existing aggregates by making a snapthot of all of them.
The new snapthot will use the new aggregate format so the definition of the old aggregate will be unnecessary after the bulk upcast and resnapshot.

You will create the shadow aggregate that mimic the old one in the same module of the current one, providing that the module is defined as rec. 

The mechanism is based on handling the failure of the Deserialization by using as fallback the Deserialization using the old aggregate and then the upcast function to the new one.