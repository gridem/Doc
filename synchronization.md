# Synchronization Ideas

## UI Synchronization

The following code snippet allows to synchronize access to a shared variable `shared_`:

```cpp
struct Guide
{
    void startSharedHandler()
    {
        go([this] {
            auto s = getNextSharedElement();
            if (s != shared_) // read access
            {
                goUi([&] { // start in UI thread
                    shared_ = s;
                }).wait(); // wait
            }
        });
    }

    Shared getSharedFromUi()
    {
        ensureUiThread();
        return shared_; // read access
    }

    Shared shared_;
};
```

The example shows that `shared_` variable is accessed from different places: from non-UI thread and from UI thread. Variable is changed only in UI thread. The construct guarantees that it cannot be read while writing because `.wait()` is used.

It looks similar to RW-locks concept but it's more interesting and not straightforward.

## Metainformation Update

To preserve data consistency one may consider the following scheme:

1. The task is to implement filesystem.
2. Filesystem contains files and metainformation.
3. Metainformation contains the folder structure, timestamps etc.

Consistent update operations:

1. Writing file to any place, it may use a lot of time.
2. On completion: atomically update metainformation related to folder structure, filesize, times etc.
3. File is appeared.

The thing is that while operation in progress there is no any long-lived locks, updates are consistent.

The same approach could be applied for hierarchy of metainformation.

## Ref Counter Update

We have intrinsic ref counter objects. To access global ref objects we have to:

1. Obtain spinlock.
2. Increase counter.
3. Release spinlock.
4. Proceed with manipulation.

The thing is that the object can be deleted on performing the following actions:

1. Load the object pointer.
2. Increase the counter.

Between load and counter increasing the object might have been deleted.

### RCU

To avoid deletion we could do the following:

1. Load the pointer. The pointer must be read by using `std::atomic`.
2. Increase the counter.

The only thing is how to prevent the object from deletion. The answer is using delayed `unref` for the counter. There are 2 options:

1. Delay `unref` itself.
2. Delay delete when `counter == 0`.

The delayed operation might be implemented using TLS on the queue of actions or event the queue of objects to be deleted or unreffed.

The idea is to use barrier to ensure that all threads are completed working with the object.

Pros is that the object will be deleted asynchronous thus preventing from unexpected delay during user request processing.

> TODO: describe the barrier.

## Transactions

The most important disadvantage of actor model is a lack of transactional semantics. The thing is that mutex allows to perform transactional semantics in the price of deadlock, e.g. one may consider the money transfer between 2 accounts. Using mutex it's possible to hold both accounts thus providing transaction while actor model is not capable to hold 2 accounts simultaniously if we consider that each account is represented by the actor. Those actors are behaved independently.

Thus to beat the actor model we should consider transactions. The idea here is simple: each object (subjector) represents state with `id` that monotonically increased on object changing (writing, mutating). We use `Alone` pattern to synchronize access to the data. But `Alone` is not capable to hold transactional updates. Thus we need to ensure that the object has the same id on returning back by comparing the `id`s. On inequality we raise an exception.

For trully transactional sematics we should avoid immediate writing subjector. Instead that we save mutations in a list and commit those mutations at once by holding (obtaining mutex) on all subjectors and checking corresponding `id`s for them.

### Optimization

`Alone` pattern dissallows concurrent reading. We should able to do that. Thus it looks more like RW-lock with shared access on read and exclusive on mutation. But because mutation itself is stored locally and not immediately applied even write could be performed concurrently in almost every cases except on commit phase.

Commit phase may use different strategies:

1. COW semantics. In that case we may perform only atomic operations on object changing.
2. Save callback to perform mutation. We mutate the object by applying the mutation. The mutation itself is applied on commit phase only.
3. Copy the object and perform mutation. On commit replace the old object with new one.

### Interface

Use may decide the following:

1. Lazy copying. On write operation if the operation returns `void` thus it can be saved for later usage and mark object as _dirty_. If the user wants to read the data we check the _dirty_ mark and apply the operation.
2. Apply the operation immediately. Lazy operation adds an overhead thus we ask the framework to apply the command explicitly.

```cpp
Transaction tx;
Subjector<MyObj> obj;
Subjector<MyObj2> obj2;
auto result = obj.performRead(); // saves the read object id inside the current transaction
obj.mutate(someData); // the object is lazilly mutated
obj2.apply().mutate(result); // the object immediately mutated
tx.commit(); // checks for all read and write ids and apply all mutations for original objects
```

#### Transaction Interface

Transactions could be:

1. _Explicit_. User specifies operation with transactions explicitly. Example: `Transaction tx; Subjector<Obj> obj; tx.at(obj).mutate();`
2. _Implicit_. User doesn't specify transaction. Transactions may be nested.

Explicit transactions allows to operate with different transactions at the same execution flow (function, coroutine etc).

Sometimes under _implicit_ transaction we need to have access to the objects without any transaction or within another explicit transaction.

### Possible Atomic Awaitings

There are several different approaches:

1. Using `EventCount` by Vyukov.
2. Using `Semaphore` by Preshing http://preshing.com/20150316/semaphores-are-surprisingly-versatile/ https://news.ycombinator.com/item?id=9212868
3. Yielding:
    - thread
    - coro
    - scheduler?

Both can be used for waitings.
