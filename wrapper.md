# Wrapper

Wrapper is further generalization of adapter.

## Idea

The basis idea is to use special processing for each function. E.g. we have the following interface:

```cpp
struct Storage
{
    Buffer read(View key);
    void write(View key, View value);
};
```

`read` method could be cached while `write` method could be batched. Thus we should use different adapters (_wrappers_) to perform those operations.

## Implementation

_Wrapper_ object could be implemented as adapter except that the interface of wrapper accepts:

1. Reference to object instance.
2. Reference to adapter state.

E.g. adapter state for `read` is a hash map, adapter state for `write` is list of handlers to be batched.

## Examples

Wrap IO async operations. Different operations may have different callbacks thus we have different wrappers.
