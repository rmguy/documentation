The Lab
=======

+Kubernetes leverages the concept of *resource versions* to achieve optimistic concurrency. All kubernetes resources have a
+"resourceVersion" field as part of their metadata. This resourceVersion is a string that identifies the internal version of an
+object that can be used by clients to determine when objects have changed. When a record is about to be updated, it's version is
+checked against a pre-saved value, and if it doesn't match, the update fails with a StatusConflict (HTTP status code 409).

