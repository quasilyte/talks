```
| Topic    | Efficient usage of Go data structures |
| Location | Kazan, FIX office                     |
| Date     | November 6, 2018                      |
```

Sub-topics:

- How arrays and slices are represented by the runtime.
- Array/slice allocations: heap vs stack.
- What escape analysis understands and its limitations.
- Subtle details about arrays and their relation to escape analysis.
- Different map container specializations in the runtime.
- Maps performance considerations.
- Some optimized map operations (and when optimizations do not apply).
- string vs []byte.
- Interfaces internals.
