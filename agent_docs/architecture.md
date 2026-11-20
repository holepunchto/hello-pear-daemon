# Foreground CLI and detached updater

> Read when changing `bin.mjs`, `App.spawnUpdater()` or how the CLI reports updater
> failures. [Agent guide](../AGENTS.md)

`bin.mjs` parses the command line once. Help and version exit early. A foreground
run with updates enabled calls `bare-daemon.spawn` via `App.spawnUpdater()` and
finishes. The hidden `--updater` branch runs the child and exits before reaching
the foreground spawn code. Preserve this ordering to avoid recursive launches.

## Source and standalone invocations

|                                             | Running from source                 | Running the standalone     |
| ------------------------------------------- | ----------------------------------- | -------------------------- |
| Executable passed to the spawner            | Bare executable                     | Installed CLI executable   |
| First child argument                        | Path to `bin.mjs`, then `--updater` | `--updater`                |
| Executable passed to `PearRuntime` as `app` | `null` (no checks or replacement)   | `os.execPath()`            |
| Default storage                             | `<tmpdir>/pear/<appName>`           | `<persistent()>/<appName>` |

The spawner always passes `--storage <dir>` and forwards an explicit update
window as a string. Forward a boolean flag as a bare token, because `paparam`
rejects a value after it. The default app name is `productName || name`.
`runUpdater()` adds `.exe` to the payload name on Windows, where
`pear-runtime-updater` accepts only `.exe` or `.msix`.

## Diagnosing startup

Use a dedicated storage directory to locate the lock, log and replicated data.
This command gives the child a one-second idle window.

```sh
npm start -- --updates --storage ./out/dev --update-window 1000
```

The foreground reports synchronous spawn errors on stderr. Once its argv
parses, the child creates the storage directory and writes update events and
errors to `./out/dev/updates.log`. Its stdio is discarded, so a child that fails
earlier, for example on a flag it does not define, creates no directory and
writes nothing to the log. A reused `./out/dev` keeps its old log.
Write child diagnostics through the `log` console in `runUpdater()`, not
`console.*`. A quiet log is normal when no update starts. Source runs never
check for one ([OTA updates](updates.md)).

To observe the child lifecycle directly, run
`npm start -- --updater --storage ./out/dev --update-window 1000`. The internal
branch explicitly enables updates and still writes diagnostics to the file.
Uncaught errors reach the terminal. Run a rebuilt standalone the same way
(`./out/<platform>-<arch>/hello-pear-daemon --updater ...`) to observe a real
check. Direct runs bypass `App.spawnUpdater()`, so after a normal launch confirm
the detached child's argv with `ps -A -o pid,ppid,args | grep -- --updater`.
