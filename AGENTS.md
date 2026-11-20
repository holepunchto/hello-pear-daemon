# AGENTS.md

Holepunch's template for short-lived Bare CLI commands with peer-to-peer OTA
updates. With updates enabled, the foreground command spawns a detached copy of
itself and exits. The child owns networking, update storage and executable replacement. The
[README](README.md) covers setup and deployment.

## Commands

```sh
npm ci
npm start                            # run the foreground CLI, updates disabled
npm start -- --version               # print the version and exit
npm start -- --updates --storage ./out/dev --update-window 1000
npm run lint                         # Prettier check and lunte
npm run format                       # write Prettier formatting
npm test                             # brittle-bare, one placeholder assertion
npm run make                         # host standalone in out/<platform>-<arch>/
npm run make:linux-x64               # Linux x64 standalone in out/linux-x64/
```

`npm start` runs with the checked-in placeholder because `--no-updates` skips
the spawn. **Set `package.json#upgrade` to a valid link from `pear touch` before
testing the updater.** The detached child parses that link. The foreground can
still print `CLI ready` and exit successfully if the child fails at startup.
Read `<storage>/updates.log` for the outcome. Standalones enable updates by default.

## Contracts

- The same `bin.mjs` handles foreground commands and hidden `--updater` invocations.
  Keep its `paparam` flags and the argv built by `App.spawnUpdater()` in sync.
- Each storage directory permits at most one active updater. Acquire the file lock before
  opening Corestore or joining the swarm. Keep it through apply and teardown.
- `--update-window` is the idle wait in milliseconds. A download that starts
  during that window can finish after it expires.
- Keep the payload basename in sync with the six `bare-build --name` arguments
  and the build workflow. Windows appends `.exe` to that basename.

## Boundaries

- Keep update work in the detached process so foreground commands can exit promptly.
- For launcher changes, check both a source run and a standalone. For lifecycle
  changes, exercise two launches against the same storage directory.
- Keep local test links out of commits. Staging, seeding and publishing a release
  are separate maintainer operations.

## Topic docs

- Changing CLI flags, spawning, storage or diagnostics → [Architecture](agent_docs/architecture.md)
- Changing locking, timeouts, update application or shutdown → [OTA updates](agent_docs/updates.md)
- Changing executable names, build scripts, CI or packaging → [Builds and releases](agent_docs/releases.md)
