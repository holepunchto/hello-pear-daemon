# Standalone builds and releases

> Read when changing the package name, `scripts/make.js`, host targets or release
> artifacts. [Agent guide](../AGENTS.md)

## Build outputs and naming

`npm run make` picks `make:<platform>-<arch>` from the current Node host.
Each target passes `--name hello-pear-daemon` to `bare-build` and writes to
`out/<platform>-<arch>/`. Supported hosts are `darwin`, `linux` and `win32` on
`arm64` and `x64`. On an Apple Silicon Mac the host build can be checked directly.

```sh
npm run make
./out/darwin-arm64/hello-pear-daemon --version
./out/darwin-arm64/hello-pear-daemon --no-updates
```

A CLI rename touches three places.

- The runtime identity in `package.json` (`productName || name`). It determines
  the default persistent storage directory and the payload basename.
- The six `--name` arguments in the target scripts.
- The upload glob, POSIX `chmod` paths and six `pear-build` inputs in
  [the build workflow](../.github/workflows/build.yaml). The Windows inputs
  include `.exe`.

## CI and the deployment directory

[`ci.yaml`](../.github/workflows/ci.yaml) lints pushes and PRs to `main`, and
builds and runs the placeholder test on six native hosts. It does not exercise
the detached updater.

The manually dispatched build workflow builds one executable per host. Its
packaging job downloads the binaries, restores the macOS and Linux executable
bits and runs `pear-build@1.1.1` with the root `package.json`. It uploads
`by-arch.tar.gz` as the `by-arch` artifact. The archive contains the deployment
directory's contents at its root. Follow the [README deployment guide](../README.md#peer-to-peer-deployments)
for staging and distribution.
