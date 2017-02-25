# Client Architecture Overview

[TOC]

## Mapkit Architecture

Based on data-driven development:

1. All data is represented as pipelined sources and destinations.
2. Coroutines read data from a channel and write data into another channel.
3. Each channel represents the data to be processed.
4. It can be done concurrently.
5. Synchronizations are inside the channels.

Cons:

1. Each coroutine requires a lot of resources.
2. Overcomplicated code because it doesn't map natively on C++ or other OOP languages.

## Subjector/Objector Architecture

1. Each objector is an object with incorporated teleport and Alone concepts. It completely eliminates deadlocks.
2. Subjector is the coroutine to switch to particular scheduler.
3. Can be isolated into different threads or thread pools.
4. It directly and natively maps on OOP languages.

## Reactive Architecture

1. Further simplifies the approach: automatically update the values, don't need to check the relations.

Cons:

1. Need to create suitable frameworks, doesn't map directly but looks like lambdas and templates allows to simplify the code.

### Example

Here is a pseudocode:

```cpp
struct Traffic
{
	react<Config> config = mapkit.getConfig();
	react<Pos> cameraPosition = map.getCameraPosition();
	react<int> regionId = getRegionId(cameraPosition);
	react<string> trafficUrl = getTrafficUrl(config);
	react<int> level = getTrafficLevel(regionId, trafficUrl);
}
```

All functions are pure and stateless e.g. they don't use the class data.

In C++:

```cpp
react<int> regionId = bindReact(getRegionId, cameraPosition);
```

## Combined Approach

Actually we need to combine react and coroutines. Coroutines allow to implement asynchronous functionality using a single function. Thus the react and coro approaches are orthogonal and *leverages* each other.

## Alternative Syntax

Here is alternative usage of reactive approach:

```cpp
rx::Obj<Config> config = mapkit.getConfig();

// asynchronous subscription due to networking
rx::Obj<Param> param = getAsync(config) + [this](Config config) {
    return network.request(config.url);
};

// synchoronous subscription
rx::Obj<Param2> param2 = get(param, config) + [this](Param param, Config config) {
    return ...;
};

// or
rx::Obj<Param2> param2 = RX_GET(param, config) {
    return ...;
};
```

## Implementation Details

Reactive objects are streams but contains only the last value (to be extracted at any time on new subscription). Subscription is invoked automatically on `get` method. Asynchronous version is important for network operations or other context switch processing.

Use may choose pull interface like this:

```cpp
rx::Obj<Config> config;
rx::Obj<Timer> timer(1s);

void myMethod()
{
    for (auto&& c: config) {
        // ...processing...
    }
}

void myMethod2()
{
    // on any change returns only changed value
    variant<Timer, Config> pc = getAny(timer, config);
    // or extract all values on any changed value
    tuple<Timer, Config> pc2 = get(timer, config);
}
```

**Note**. Values can be shared across different threads. E.g the function:

```cpp
Obj<Config> getConfig();
```

Returns `shared_ptr` for the same instance on any time. You don't need to create different shared_ptr's.
