# Feedback on Go plugins for hot code swapping demo

What I wanted to build was a binary that would reguarily run the .so
files found in a given directory. When one of those .so files changes
I would have liked to run the new version.

My ideal demo script was:

```bash
$ go build --buildmode=plugin plugin.go  # generates plugin.so
$ go run main.go  # loads plugin.so and runs its Run function.
Hello, plugin!

# from a different terminal
$ vi plugin.go  # replace "Hello, plugin!" with "Bye, plugin!".
$ go build --buildmode=plugin plugin.go  # generates a new version of plugin.go

# go back to terminal running main.go
Hello, plugin!  # from previous execution
Bye, plugin!  # new version is loaded
```

This didn't work, and instead got `"Hello, plugin!"` all the time,
as described by the documentation of `plugin.Open`.

## But I want to load a new one!

I then tried changing the name of the .so file at each compilation,
with the hopes that would avoid the caching mechanism.

```bash
$ go build --buildmode=plugin -o v1.so plugin.go  # generates v1.so
$ go run main.go  # loads v1.so and runs its Run function.
Hello, plugin!

# from a different terminal
$ vi plugin.go  # replace "Hello, plugin!" with "Bye, plugin!".
$ go build --buildmode=plugin -o v2.so plugin.go  # generates v2.so

# go back to terminal running main.go
Hello, plugin!  # from previous execution

plugin: plugin plugin/unnamed-304e4b1e7984e8373d868f9602a7b6bf865b1cb9 already loaded
fatal error: plugin: plugin already loaded

goroutine 1 [running]:
runtime.throw(0x58467f, 0x1d)
	/usr/local/go/src/runtime/panic.go:596 +0x95 fp=0xc420045978 sp=0xc420045958
plugin.lastmoduleinit(0x22f9400, 0xc4200bead8, 0x22fb100, 0x25, 0x826f60)
	/usr/local/go/src/runtime/plugin.go:25 +0xc1a fp=0xc420045aa8 sp=0xc420045978
plugin.open(0xc42006a660, 0x25, 0x0, 0x0, 0x0)
	/usr/local/go/src/plugin/plugin_dlopen.go:72 +0x25d fp=0xc420045ce8 sp=0xc420045aa8
plugin.Open(0xc42006a660, 0x25, 0xc420045d88, 0xc420014660, 0x58)
	/usr/local/go/src/plugin/plugin.go:30 +0x35 fp=0xc420045d20 sp=0xc420045ce8
main.(*loader).call(0xc42000c160, 0xc42006a660, 0x25, 0x25, 0x25)
	/data/main.go:131 +0x4d fp=0xc420045e18 sp=0xc420045d20
main.(*loader).compileAndRun(0xc42000c160, 0xc420010610, 0x6, 0x0, 0x0)
	/data/main.go:88 +0x204 fp=0xc420045ee8 sp=0xc420045e18
main.main()
	/data/main.go:49 +0x152 fp=0xc420045f88 sp=0xc420045ee8
runtime.main()
	/usr/local/go/src/runtime/proc.go:185 +0x20a fp=0xc420045fe0 sp=0xc420045f88
runtime.goexit()
	/usr/local/go/src/runtime/asm_amd64.s:2197 +0x1 fp=0xc420045fe8 sp=0xc420045fe0

goroutine 17 [syscall, locked to thread]:
runtime.goexit()
	/usr/local/go/src/runtime/asm_amd64.s:2197 +0x1
exit status 2
```

Now this was unexpected, and it is a non documented behavior.

## So what's the cache key?

I assumed the cache key was the path of the `.so` file path, but after this I started
thinking it has to be something else: let's see if it's the `.go` path?

```bash
$ go build --buildmode=plugin -o v1.so plugin.go  # generates v1.so
$ go run main.go  # loads v1.so and runs its Run function.
Hello, plugin!

# from a different terminal
$ cp plugin.go pluginv2.go 
$ vi pluginv2.go  # replace "Hello, plugin!" with "Bye, plugin!".
$ go build --buildmode=plugin -o v2.so pluginv2.go  # generates v2.so

# go back to terminal running main.go
Hello, plugin!  # from previous execution of v1.so
Bye, plugin!    # from v2.so
Hello, plugin!  # from v1.so
...
```

That's what my program now does, but it feels very wrong to have a continuously
growing cache of plugins.

## Questions

Some questions from me and the audience at FOSDEM.

- Is there any plans to provide a `func (p *Plugin) Release()` or similar?
- Is the `panic` an issue in the code that should be fixed, or missing docs?
- What are the plans to provide this on Windows/Mac?
- What happens when two different plugins both include the same lib?
  - Do they repeat/share the symbols?
  - What if they provide different/incompatible versions of the same library?
