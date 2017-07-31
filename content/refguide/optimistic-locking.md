---
title: "Optimistic Locking"
parent: "data-storage"
---

# Optimistic Locking

## Introduction
_Optimistic Locking_ is a technique to guarantee prevention for concurrent modification of a table record. It differs from _pessimistic locking_ as the latter puts an actual lock on the database record at the time of reading. _Pessimistic Locking_ reduces the performance and scalability of the system. Also, this is not an applicable pattern for stateless services. _Optimistic locking_ uses a `version` column in the table to verify whether the record is still the same as it was at the time of reading it.

## Available since
Optimistic locking is available from Mendix 7.6 onwards for everything except the client. That means that all updates and deletes triggered from the client will _not_ be guarded by optimistic locking. This also applies to objects send as input for microflow actions. Support for optimistic locking using the client will be implemented in future editions of Mendix.

## Behavior before Mendix 7.6 or when Optimistic Locking is disabled
Before Mendix 7.6 the runtime did no locking. Concurrent Modifications are resolved by the Last Writer Wins strategy. This is still the behavior when optimistic locking is disabled in the `Runtime` tab of the `Project Settings`.

## Implementation in Mendix
When Optimistic Locking is enabled, each entity gets an additional system attribute with the name `MxObjectVersion` of type `Long`. This field is automatically populated with the correct value. The default value is `1` and this value will be automatically increased every commit of that entity instance.

Upon _update_ and _delete_, the attribute value is read from the object and compared to the value available for this record in the database. If it is the same, then the update or delete will proceed. If it is different, a `ConcurrentModificationRuntimeException` is thrown, preventing the update or delete from proceeding. The `MxObjectVersion` attribute on the Mendix object is not write-protected. Setting this value however will not result in this value being saved into the database. It's current value will be used to compare it with the value for the same record in the database.

### Limitations of the current implementation
The current implementation works on all changes initiated and committed within one backend request. The client part is not done yet, meaning that objects send from the client don't preserve the version yet. As such in changes done from the client it is still the case that the last writer wins. 

### New projects and migration
From Mendix 7.6 onwards, Optimistic Locking is enabled by default when you create a new app. However, when migrating existing apps to Mendix 7.6, this feature is disabled by default.

In case an existing app already had the `MxObjectVersion` attribute, then a duplicate attribute will be reported in the Modeler. This must be fixed by renaming the existing attribute to another name. The system attribute cannot be renamed.

### Impact on insert
There is no impact on insert as this will just introduce the record in the database.

### Impact on update and delete
When no concurrent modification occurs, update or delete happens as before. However, when a concurrent modification is detected, the runtime will throw a `ConcurrentModificationRuntimeException`. This exception prevents the transaction from succeeding. If the entity instance was deleted before applying the update or the delete, then this will result in an exception stating that the record cannot be found (as is the case without optimistic locking too).

### Impact on performance
This attribute is only added to the entities that are not derived from other entities. This way all entities will have this attribute (the derived entities will derive it from the parent entity). This causes every entity to have maximum one extra attribute in queries and an extra check upon update or delete. There can be some performance impact, although it is expected to be minor.

### How the error is reported
The error reported in the client is "An error has occurred during processing the request". The runtime log file contains an entry with the following details:
```
[..] 
com.mendix.systemwideinterfaces.connectionbus.data.ConcurrentModificationRuntimeException: Object of type 'MyFirstModule.MyEntity' with guid '3940649673949185' cannot be updated, as it is modified by someone else
	at MyFirstModule.MyMicroflow (Change : 'Change 'MyEntity'')
[..]
```

This error shows that there was a `ConcurrentModificationRuntimeException` during execution of the change action `Change 'MyEntity'` of the microflow `MyFirstModule.MyMicroflow. The object had the id `3940649673949185` and was of type `MyFirstModule.MyEntity`.

### Solving `ConcurrentModificationRuntimeException` errors
Solving the `ConcurrentModificationRuntimeException` error depends on the action. If it happend during a commit action from the client: retry the commit. If it happened during the microflow execution: retry execution of the microflow. If it fails again, then most probably the microflow contains a construct that triggers this (this can happen when the same object is loaded twice and both instances get modified and committed). Solve that situation to solve the `ConcurrentModificationRuntimeException`.

### Customize handling of `ConcurrentModificationRuntimeException` in microflows
On the actions that commits new entity instances or updates, you can set the _Error Handling_ to either `Custom with rollback` or `Custom without rollback`. Then create an outgoing flow and set it as _error handler_. On this flow you now can use the `$latestError` variable to check it's `ErrorType`. If it equals `com.mendix.systemwideinterfaces.connectionbus.data.ConcurrentModificationRuntimeException` then this is a concurrent modification exception.

#### Customization with `Custom without rollback`
When choosing the errorhandling as `Custom without rollback`, the changes done earlier in the microflow are not rolled back. Continuation means that the object on which the `ConcurrentModificationRuntimeException` was thrown can not be used anymore and should be reloaded from the database if you want to make additional changes to that object. Note that only the changes done in that _Change_, _Delete_ or _Commit_ action are no longer available. All other changes will still be committed to the database.

#### Customization with `Custom with rollback`
When choosing the errorhandling as `Custom with rollback`, the changes up to that point are rolled back. A successful continuation after this error being handled will only commit the changes made after this point. The object causing the `ConcurrentModificationRuntimeException` needs to be reloaded as otherwise it will cause the same exception again when committed.
