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

A CLI rename must keep three places matching. All three spell `productName || name`
in lowercase kebab-case. `bare-build` kebab-cases `--name` for the file name,
and `pear-build` rejects an app whose basename differs.

- The runtime identity in `package.json` (`productName || name`). It determines
  the default persistent storage directory and the payload basename.
- The six `--name` arguments in the target scripts.
- The upload glob, POSIX `chmod` paths and six `pear-build` inputs in
  [the build workflow](../.github/workflows/build.yaml). The Windows inputs
  include `.exe`.

CI and `npm run make` do not check names, and the build workflow misses some
mismatches. `pear-build` checks input basenames, not that the files exist, so a
wrong Windows path packages an empty `app/`. List `by-arch.tar.gz` before
staging. `bare-build` does not clear `out/`, so delete the old executable before
checking the new one. When `name` changes, run `npm install --package-lock-only`.
Rename before the first release. Existing installs look up
`/by-arch/<platform>-<arch>/app/<old name>` and log `update not found` once a
release carries only the new name.

## CI and the deployment directory

[`ci.yaml`](../.github/workflows/ci.yaml) lints pushes and PRs to `main`, and
builds and runs the placeholder test on six native hosts. It does not exercise
the detached updater.

The manually dispatched build workflow builds one executable per host. Its
packaging job downloads the binaries, restores the macOS and Linux executable
bits and runs `pear-build@1.1.1` with the root `package.json`. It uploads
`by-arch.tar.gz` as the `by-arch` artifact. The archive contains the deployment
directory's contents at its root.

Each binary embeds `version` and `upgrade` from the dispatched ref's
`package.json`, and `pear-build` copies that file to the deployment root. A
maintainer commits the release `upgrade` link and a higher `version` before
dispatching. Follow the
[README deployment guide](../README.md#peer-to-peer-deployments) for staging and
distribution.
