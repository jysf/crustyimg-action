# crustyimg-action

Run [`crustyimg`](https://github.com/jysf/crustyimg) in CI to **lint an image asset tree** — a
format-aware, no-URL check that flags GPS/EXIF leaks, over-budget or wrong-format assets,
non-baked orientation, corrupt files and more — and turns each finding into an inline PR
annotation with a runnable fix. An `optimize` mode is included too.

It installs crustyimg via [`setup-crustyimg`](https://github.com/jysf/setup-crustyimg), then runs
`crustyimg lint … --format json` and emits native `::error`/`::warning`/`::notice` annotations plus
a job-summary table.

## Usage

```yaml
name: images
on: [pull_request]
jobs:
  lint-images:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: jysf/crustyimg-action@v1
        with:
          paths: assets content        # what to lint (default: the whole repo)
```

A GPS leak or an over-budget asset fails the job (exit 7) and annotates the offending file; a clean
tree passes. Configure rules with a `.crustyimg-lint.toml` in your repo, or via `args`.

### Optimize mode

```yaml
      - uses: jysf/crustyimg-action@v1
        with:
          mode: optimize
          paths: assets
          args: --out-dir optimized
```

## Inputs

| Input        | Default  | Description |
|--------------|----------|-------------|
| `mode`       | `lint`   | `lint` (annotate findings) or `optimize` (re-encode a tree). |
| `paths`      | `.`      | Files, directories, or globs to check (space-separated). |
| `args`       | *(empty)*| Extra args passed to crustyimg (e.g. `--select privacy`, `--config path`, `-o`). |
| `version`    | `latest` | crustyimg release to install (passed to `setup-crustyimg`). |
| `fail-level` | `error`  | `error` (fail on error findings), `warn` (also fail on warnings), or `never` (report only). |
| `install`    | `true`   | Install crustyimg via `setup-crustyimg`. Set `false` if it's already on `PATH`. |

## What it emits

- **Annotations:** one per finding — `error` → `::error`, `warn` → `::warning`, `info` → `::notice`
  — anchored to the file, titled `crustyimg <rule-id>`, with the runnable fix in the message.
- **Job summary:** a counts line + a table of findings.
- **Exit code:** propagates crustyimg's gate (`0` clean · `7` when the gate trips), honoring
  `fail-level`.

## Future work (not built here)

- **Autofix / commit-back mode** — running the suggested fixes and committing them back to the PR —
  is deferred to a v2. It needs write permissions and careful fork-safety handling (a PR from a fork
  can't be granted write to the base repo), so it is intentionally out of scope for v1.

## Maintainer / release notes

- `crustyimg lint` ships in **crustyimg 0.4.0+** (published to crates.io / Homebrew / GitHub
  Releases). This action installs it via [`jysf/setup-crustyimg@v1`](https://github.com/jysf/setup-crustyimg).
- Tagged **`v1`** (moving major) + `v1.0.0`. A GitHub Marketplace listing (optional) is via
  *Releases → Publish this Action* — the maintainer's step.

## License

MIT OR Apache-2.0, matching crustyimg.
