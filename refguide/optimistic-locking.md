---
title: "Optimistic Locking"
space: "Mendix 7 Reference Guide"
parent: "data-storage"
---

# Optimistic Locking

## Introduction
Optimistic Locking is a technique to guarantee prevention for concurrent modification of a table record. It differs from _pessimistic locking_ as the latter puts an actual lock on the database record at the time of reading. This reduces the performance and scalability of the system. Also, this is not an applicable pattern for stateless services. Optimistic locking uses a `version` column in the table to verify whether the record is still the same as it was at the time of reading it.

## Available since
Optimistic locking is available from Mendix 7.5 onwards.

## Behavior before Mendix 7.5 or when Optimistic Locking is disabled
Before Mendix 7.5 the runtime did no locking. Concurrent Modifications are resolved by the Last Writer Wins strategy. This is still the behavior when optimistic locking is disabled in the `Runtime` tab of the `Project Settings`.

## Implementation in Mendix
When Optimistic Locking is enabled, each entity gets an additional system attribute with the name `MxObjectVersion` of type `Long`. This field is automatically populated with the correct value. The default value is `1` and this value will be automatically increased every commit of that entity instance. 

### New projects and migration
From Mendix 7.5 onwards, Optimistic Locking is enabled by default when you create a new app. However, when migrating existing apps to Mendix 7.5, this feature is disabled by default.

In case an existing app already had the `MxObjectVersion` attribute, then a duplicate attribute will be reported in the Modeler. This must be fixed by renaming the existing attribute to another name. The system attribute cannot be renamed.

### Impact on insert
There is no impact on insert as this will just introduce the record in the database.

### Impact on update and delete
When no concurrent modification happens, update or delete happens as before. However, when a concurrent modification is detected, the runtime will throw a `ConcurrentModificationException`. This exception prevents the transaction from succeeding. If the entity instance was deleted before applying the update or the delete, then this will result in an exception stating that the record cannot be found (as is the case without optimistic locking too).

### Impact on performance
This attribute is only added to the entities that are not derived from other entities. This way all entities will have this attribute (the derived entities will derive it from the parent entity). This causes every entity to have maximum one extra attribute in queries and an extra check upon update or delete. There can be some performance impact, although it is expected to be minor.

### Setting the `MxObjectVersion` attribute
The `MxObjectVersion` attribute is not write-protected. This allows this attribute to be set during execution. However, setting this attribute might result in a `ConcurrentModificationException` in case the value does not correspond with the value in the database. This allows this feature to be used for integration purposes.

