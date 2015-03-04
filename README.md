The Lab
=======

Kubernetes leverages the concept of *resource versions* to achieve optimistic concurrency. All kubernetes resources have a
"resourceVersion" field as part of their metadata. This resourceVersion is a string that identifies the internal version of an
object that can be used by clients to determine when objects have changed. When a record is about to be updated, it's version is
checked against a pre-saved value, and if it doesn't match, the update fails with a StatusConflict (HTTP status code 409).


The resourceVersion is changed by the server every time an object is modified. If resourceVersion is included with the PUT operation
the system will verify that there have not been other successful mutations to the resource during a read/modify/write cycle, by
verifying that the current value of resourceVersion matches the specified value.

The resourceVersion is currently backed by [etcd's modifiedIndex](https://coreos.com/docs/distributed-configuration/etcd-api/).
However, it's important to note that the application should *not* rely on the implementation details of the versioning system
maintained by kubernetes. We may change the implementation of resourceVersion in the future, such as to change it to a timestamp or
per-object counter.
