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
3. Once ready, the daemon starts the idle timer (default 30 seconds) if it is
   still waiting and has not seen `updating`. `--update-window <ms>` accepts a
   non-negative safe integer, including zero. An `updating` event clears the
   timer so a slow transfer can finish. Nothing bounds the transfer after that.
   An equal-version prefetch emits no `updating`, so the timer keeps running.
4. `updated` triggers apply. Completion comes from `update-applied`, an error,
   close or the idle timer. `runUpdater()` then closes the app and log. Teardown
   clears the timer, closes the swarm, runtime and store, and releases the lock
   in a `finally` block.

Teardown has two known gaps:

- Closing the runtime awaits any in-flight check, prefetch or download, which
  stalls once the swarm is destroyed. The process then exits on an empty event
  loop with teardown unfinished. During a download it hangs with the lock held
  until SIGKILL, including after SIGTERM or SIGINT.
- Teardown does not await `_applyUpdate()`. A signal or unrelated error during
  apply can release the lock and reach `Bare.exit()` before apply finishes. On
  Windows that can interrupt the copy-and-rename sequence.

## Checking a release

The locked `pear-runtime-updater` 3.4.0 starts automatic checks only in a bundled
app, meaning `PearRuntime` received a non-null `app`. A source run with
`--updates` exercises spawning, locking and replication. Use a standalone built
with a valid upgrade link to exercise download and apply.

The standalone checks once at startup against cached drive state, then again
when replication appends to the drive. That recheck is immediate within 60
seconds of updater start, then delayed randomly by up to an hour. `updating`
must start before the idle timer fires, so with `--update-window` above 60000 a
release that arrives after 60 seconds is usually missed.

Each check compares the drive-root `package.json#version` against the embedded
version. A strictly higher version and a matching
`/by-arch/<platform>-<arch>/app/<name>` are required. A higher version without
that payload logs `update not found` and ends the run. An equal version can
prefetch blocks without emitting `updating`.

Stage each payload with the `package.json` it was built from. If the drive
version is higher than the payload's embedded version, every launch downloads
and applies it again.

The daemon logs download deltas and applies the downloaded binary automatically.
On macOS and Linux the updater swaps the executable. On Windows it copies the
incoming `.exe` alongside the installed one, renames the old file and moves the
new version into place. Run the command again with `--version` to check which
version starts.
