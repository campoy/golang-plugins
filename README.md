# golang-plugins

golang-plugins uses the new plugin feature of Go 1.8 to
implement hot code swapping in Go.

This is highly experimental and just a way for me to learn
how plugins work and what limitations I find.

Run this program and then try editing, adding, or removing
files in the `plugins` directory.

```bash
go run main.go
```

The entry point to the plugins is the `Run` function, that
doesn't receive any parameters and returns an error.

## Limitations:

- This only works on Linux.
- We poll regularly the plugins directory instead of using fsnotify.
- We recompile every time, even if the code has not changed.
- This causes a continuously growing memory requirement (memory leak?).

### Disclaimer

This is not an official Google product (experimental or otherwise), it is just
code that happens to be owned by Google.