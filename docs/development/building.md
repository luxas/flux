# Building Weave Flux

You'll need a working `go` environment version >= 1.11 (official releases are built against `1.12`).
It's also expected that you have a Docker daemon for building images.

Clone the respository. The project uses Go Modules, so if you explicitly define `$GOPATH` you should
clone somewhere else

Then, from the root directory,

```sh
$ make
```

This makes Docker images, and installs binaries to `$GOBIN` (if you define it) or `$(go env GOPATH)/bin`.

Note: The default target architecture is amd64. If you would like to try to build Docker images
and binaries for a different architecture you will have to set ARCH variable,

```sh
$ make ARCH=<target_arch>
```

## Running tests

```sh
$ make test
```
