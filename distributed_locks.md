# Distributed Locks Application

## Architecture

All data are placed into the map or a set of maps.

Any entity has the following configuration:

1. TTL - how much time it preserves in the storage.
2. Action on destroy.
3. Parent entity.
4. Children entities.

When parent uses the children it automatically pinged.
When parent is destroyed it destroyes the temporary objects (view typically).

## Use Cases

Use may use the following use cases.

### Simple Locking

User tries to obtain the lock. If it receives OK on lock => lock is obtained. It must periodically ping the lock object.

There are 2 objects have been created:

1. The lock itself with TTL. It can be like 1 day or 1 week.
2. The view on lock object to be pinged. TTL must be like 10 sec or 1 min (or even 1 sec). The lock view is the lock guard. When the lock guard is expired the lock is released automatically.

Lock view contains generation of the lock: monotonically increased counter.

### Sessing Locking

User creates the session and service returns the `session_id`. Using this session it tries to obtain lock. If the disconnection happen it may reconnect using `session_id` and checks for obtained locks. Simple locking scheme disallows to check the lock on disconnection => use must wait on unlock on TTL expiration. 

### Queue

Use creates the producer or consumer view on queue. The view contains the offset of read from or write to the object. On successful write operation (produce) the service atomically increase the offset of the view and put the data into the queue.

## Read Consistency

To speedup read consistency let's do the following trick. On connection we sync the version based on majority and try to obtain the latest data. After that we are listening for the data based on data from that node obtaining the sequential consistency.

There is must be option to use linearizable reads by syncing the data.
