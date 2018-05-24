```
| Topic    | Binary size optimizations in Go |
| Location | Innopolis University            |
| Date     | May 31, 2018                    |
```

Sub-topics:

- Reasons behind fat binaries produced by Go toolchain.
- Why it's impossible to merge duplicated blocks for the optimizer.
- Other toolchain components that may fail to perform size optimizations (like linker).
- Currently working and (almost) safe ways of shrinking Go binaries.
- Case-study of [CL107895](https://golang.org/cl/107895).
