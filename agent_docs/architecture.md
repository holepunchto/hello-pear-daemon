# Foreground CLI and detached updater

> Read when changing `bin.mjs`, `App.spawnUpdater()` or how the CLI reports updater
> failures. [Agent guide](../AGENTS.md)

`bin.mjs` parses the command line once. Help and version exit early. A foreground
run with updates enabled calls `bare-daemon.spawn` via `App.spawnUpdater()` and
finishes. The hidden `--updater` branch runs the child and exits before reaching
the foreground spawn code. Preserve this ordering to avoid recursive launches.

## Source and standalone invocations

|                                                    | Running from source                 | Running the standalone     |
| -------------------------------------------------- | ----------------------------------- | -------------------------- |
| Executable passed to the spawner                   | Bare executable                     | Installed CLI executable   |
| First child argument                               | Path to `bin.mjs`, then `--updater` | `--updater`                |
| Executable passed to `PearRuntime` for replacement | `null`                              | `os.execPath()`            |
| Default storage                                    | `<tmpdir>/pear/<appName>`           | `<persistent()>/<appName>` |

The spawner always passes `--storage <dir>` and forwards an explicit update
window as a string. The default app name is `productName || name`. The updater
adds `.exe` to its payload name on Windows.

## Diagnosing startup

Use a dedicated storage directory to locate the lock, log and replicated data.
This command gives the child a one-second idle window.

```sh
npm start -- --updates --storage ./out/dev --update-window 1000
```

The foreground reports synchronous spawn errors on stderr. The child creates
the storage directory and writes update events and runtime errors to
`./out/dev/updates.log`. A quiet log is normal when no update starts.
To observe the child lifecycle directly, run
`npm start -- --updater --storage ./out/dev --update-window 1000`. The internal
branch explicitly enables updates and still writes diagnostics to the file.
