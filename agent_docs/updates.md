# Updater lifetime and OTA behavior

> Read before changing `App.updater()`, its timer, resource ownership or the
> apply path in `app.js`. [Agent guide](../AGENTS.md)

## Lock and idle window

1. The daemon opens `<storage>/updater.lock` and tries to lock its file descriptor.
   On contention it closes the descriptor and returns before opening Corestore.
   The lock file can remain on disk after exit. Its presence alone says nothing
   about whether an updater is running.
2. The lock holder registers completion listeners before calling `ready()`.
   It opens `<storage>/pear-runtime/corestore`, then uses its Hyperswarm to
   replicate the drive as a client. A reachable peer must seed the release.
3. Once ready, the daemon starts a 30-second idle timer if it is still waiting
   and no download is in progress. `--update-window <ms>` accepts a non-negative
   safe integer, including zero. An `updating` event clears the timer so a slow
   transfer can finish.
4. `updated` triggers apply. Completion comes from `update-applied`, an error,
   close or the idle timer. The launcher then closes the app and log. Teardown
   clears the timer, closes the swarm, runtime and store, and releases the lock
   in a `finally` block.

## Checking a release

The locked `pear-runtime-updater` 3.4.0 starts automatic checks only in a bundled
app. A source run with `--updates` exercises spawning, locking and replication.
Use a standalone built with a valid upgrade link to exercise download and apply.

The standalone checks on startup. It compares the drive-root `package.json#version`
against its embedded version. A strictly higher version and a matching
`/by-arch/<platform>-<arch>/app/<name>` are required. An equal version can
prefetch blocks without emitting `updating`.

The daemon logs download deltas and applies the downloaded binary automatically.
On macOS and Linux the updater swaps the executable. On Windows it copies the
incoming `.exe` alongside the installed one, renames the old file and moves the
new version into place. Run the command again with `--version` to check which
version starts.
