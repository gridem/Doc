# Online Data Processing

The article is about online data processing. It's kind of realtime data analysis but it may involve complex logic including reducers (along with mappers) and those reducers can be stateful.

We have to have a functionality to do fault-tolerant online processing.

## Subscriptions

The idea is to have a functionality to subscribe for the channel. E.g. some service outputs some data and we would like to listen those values for the set of nodes:

```cpp
template<T_values>
struct Subscriber
{
    void subscribe(set<NodeId>);
};
```

## Model

Currently we have model with `std::function<void()>` broadcasting. Let's extent this model to send an values:

```cpp
struct Channel<T_value>
{
    void add(T_value value);
}
```

We broadcast the invocation of `Channel<T>::add` to the subscribers (receivers) and they apply the `std::function<void()>` with `::add` invocation.

## Replication

In order to preserve the object we need to replicate it. The replication is a subscription for the object changes.

There are 2 possibilities:

1. Replicate input stream.
2. Replicate internal state.

The thing is that we need to save our CPU. Our object does CPU intensive operations thus we would like to save this CPU resources. In addition to that internal state usually requires less memory than input stream. Thus the second option is prefferable.

## Distributed Analogy

To reason about distributed systems it's worth to introduce analogy based on single process software development.

Each service represents an object that user can operation.

If system has dynamic configuration e.g. routing to service it's necessary to introduce another object `Router` that handles routes to that object. Because the route can be changed at any time you need to have "effectively" obtain 2 locks: for `Router` and for the object.

But this task could be resolved using optimistic locking scheme:

1. Client has `version_id` related to the `Router` instance. `version_id` is like a reference to `Router` object (or e.g. copy-on-write value).
2. Client sends request to the object with included `version_id`. If server has greater version it responds with appropriate message to repeat the request with updated `version_id`.

Thus we need to hold reference to `Router` and mark them as dependent instances. But it should be transparent to the user: user shouldn't think about that. One of the possibility is to have global version for routes and send it to participants in any request and handle it in the single place for any call.
