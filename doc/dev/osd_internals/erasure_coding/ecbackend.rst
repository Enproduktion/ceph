=================================
ECBackend Implementation Strategy
=================================

Misc Design Notes
=================


- CEPH_OSD_OP_APPEND: We can roll back an append locally by
  including the previous object size as part of the PG log event.
- CEPH_OSD_OP_DELETE: The possibility of rolling back a delete
  requires that we retain the deleted object until all replicas have
  persisted the deletion event.  ErasureCoded backend will therefore
  need to store objects with the version at which they were created
  included in the key provided to the filestore.  Old versions of an
  object can be pruned when all replicas have committed up to the log
  event deleting the object.
- CEPH_OSD_OP_(SET|RM)ATTR: If we include the prior value of the attr
  to be set or removed, we can roll back these operations locally.

Core Changes:

- Current code should be adapted to use and rollback as appropriate
  APPEND, DELETE, (SET|RM)ATTR log entries.
- The filestore needs to be able to deal with multiply versioned
  hobjects.  This means adapting the filestore internally to
  use a `ghobject <https://github.com/ceph/ceph/blob/firefly/src/common/hobject.h#L238>`_ 
  which is basically a tuple<hobject_t, gen_t,
  shard_t>.  The gen_t + shard_t need to be included in the on-disk
  filename.  gen_t is a unique object identifier to make sure there
  are no name collisions when object N is created +
  deleted + created again. An interface needs to be added to get all
  versions of a particular hobject_t or the most recently versioned
  instance of a particular hobject_t.
