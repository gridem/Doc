# Coroutines Discussion

This document relates to discussion: stackless coroutines vs stackful coroutines.

Why not stackless coroutines:

1. No dtor support
2. Cancer a()->b()->c()->d() if d() becomes the async thus a must be async
3. Heap allocation for any call from the previous
4. Call switch on any invocation introduces overhead
4. std Algorithms must be rewritten
5. async lambdas?
6. threads and coroutines are not the same completely

Thus performance and memory is debatable due to 3 and 4
