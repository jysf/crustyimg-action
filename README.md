# crustyimg-action

Run [`crustyimg`](https://github.com/jysf/crustyimg) in CI to **lint an image asset tree** — a
format-aware, no-URL check that flags GPS/EXIF leaks, over-budget or wrong-format assets,
non-baked orientation, corrupt files and more — and turns each finding into an inline PR
annotation with a runnable fix. An [`optimize` mode](#optimize-mode) shrinks a tree of images
instead, for a build or deploy step.

It installs crustyimg via [`setup-crustyimg`](https://github.com/jysf/setup-crustyimg), then runs
`crustyimg lint … --format json` and emits native `::error`/`::warning`/`::notice` annotations plus
a job-summary table.

**New to this?** The [setup guide](https://github.com/jysf/crustyimg/blob/main/docs/github-actions.md) walks through budgets, required checks,
pull requests from forks and fixing findings, and [`jysf/crustyimg-example`](https://github.com/jysf/crustyimg-example) is a live repo
using it ([pull request #1](https://github.com/jysf/crustyimg-example/pull/1) shows the annotations).

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
          paths: assets/**/* content/**/*   # what to lint (default: images at the repo root)
```

A GPS leak or an over-budget asset fails the job (exit 7) and annotates the offending file; a clean
tree passes. Configure rules with a `.crustyimg-lint.toml` in your repo, or via `args`.

A directory in `paths` covers only the images directly inside it. To include subfolders, use a
recursive glob such as `assets/**/*`. The action passes globs to crustyimg as written, so you
don't need to quote them.

## Optimize mode

`mode: optimize` runs `crustyimg optimize` over `paths` and writes the results to a folder. Use it
in a build or deploy step, to ship smaller images without changing the ones you commit.

```yaml
      - uses: jysf/crustyimg-action@v1
        with:
          mode: optimize
          paths: assets/**/*
          args: --out-dir _site/assets --max 1600
```

For each image, `optimize`:

- turns it upright and removes its metadata, including any GPS location,
- re-encodes it and keeps the smallest result, often a different format: a `photo.jpg` usually
  comes out as `photo.avif`,
- never writes a file larger than the original. If it can't do better, it copies the original.

It keeps the image's dimensions unless you pass `--max`. Your source files are never changed.

### Useful `args`

| `args` | What it does |
|---|---|
| `--out-dir <dir>` | Where to write the results. Required when `paths` matches more than one file. |
| `--max 1600` | Also shrink anything wider or taller than 1600 px. |
| `--profile preserve` | Keep each image's format and file name, so `photo.jpg` stays `photo.jpg`. Use this when your pages link to the original names. |
| `--name-template '{stem}.min.{ext}'` | Name the outputs yourself. |
| `--max-size 200KB` | Re-encode each image to fit a byte budget. |
| `--target high` | Search for the smallest file that still looks right, instead of a fixed quality. Slower. |
| `-o <file>` | Write a single input to that file instead of a folder. |

See `crustyimg optimize --help` for the rest.

### Things to know

- **The results stay on the runner.** The action doesn't commit them back. Deploy or upload the
  output folder in a later step, for example with `actions/upload-pages-artifact` or
  `actions/upload-artifact`.
- **`--out-dir` puts every output directly in that folder**, without its subfolders. Two images
  with the same name in different subfolders fail the step (exit 6). Run the action once per folder
  to keep them apart.
- **A pattern that matches nothing fails the step** (exit 3).
- **There are no annotations or job summary in this mode**, and `fail-level` has no effect. The
  step fails only if crustyimg can't process an image.

### When to use `setup-crustyimg` instead

The mode runs `optimize` and nothing else. For any other command, install crustyimg with
[`jysf/setup-crustyimg`](https://github.com/jysf/setup-crustyimg) and call it in a `run:` step:

- `crustyimg web` resizes every image to a fixed size (2048 px by default) as well. Your pages then
  need to link to the new file names.
- `crustyimg build` runs your own recipes, such as a watermark or an exact-size social card,
  from a `crustyimg.build.toml`.

The [setup guide](https://github.com/jysf/crustyimg/blob/main/docs/github-actions.md#shrink-images-when-you-deploy)
shows both, and [`jysf/crustyimg-example`](https://github.com/jysf/crustyimg-example) uses `build`
to deploy its site.

## Inputs

| Input        | Default  | Description |
|--------------|----------|-------------|
| `mode`       | `lint`   | `lint` (annotate findings) or [`optimize`](#optimize-mode) (write smaller copies). |
| `paths`      | `.`      | Files, directories, or globs to lint or optimize (space-separated). A directory covers only its top-level images; use `dir/**/*` for subfolders. |
| `args`       | *(empty)*| Extra args passed to crustyimg (e.g. `--select privacy`, `--config path`, `-o`). |
| `version`    | `latest` | crustyimg release to install (passed to `setup-crustyimg`). |
| `fail-level` | `error`  | Lint only. `error` (fail on error findings), `warn` (also fail on warnings), or `never` (report only). |
| `install`    | `true`   | Install crustyimg via `setup-crustyimg`. Set `false` if it's already on `PATH`. |

## What lint mode emits

- **Annotations:** one per finding — `error` → `::error`, `warn` → `::warning`, `info` → `::notice`
  — anchored to the file (as a path relative to the repo, so it shows inline on the PR), titled `crustyimg <rule-id>`, with the runnable fix in the message.
- **Job summary:** a counts line + a table of findings.
- **Exit code:** propagates crustyimg's gate (`0` clean · `7` when the gate trips), honoring
  `fail-level`.

## Future work (not built here)

- **Autofix / commit-back mode** — running the suggested fixes and committing them back to the PR —
  is deferred to a v2. It needs write permissions and careful fork-safety handling (a PR from a fork
  can't be granted write to the base repo), so it is intentionally out of scope for v1.

## Requirements & versioning

- `crustyimg lint` ships in **crustyimg 0.4.0+**; the action is tested against **0.8.0** and
  `latest` on Linux, macOS and Windows. The `version` input defaults to `latest`;
  the action installs the CLI via
  [`jysf/setup-crustyimg`](https://github.com/jysf/setup-crustyimg), which verifies the release
  checksum.
- `v1` is a moving major tag — it always points at the newest `v1.x` release, so
  `uses: jysf/crustyimg-action@v1` picks up fixes without a workflow edit. Pin an exact release
  (`@v1.0.3`) if you'd rather control upgrades yourself.

## License

Dual-licensed under either [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE) at your
option, matching crustyimg.
