# Contributing

## Set up development

Install the [prerequisites listed in the README](README.md#prerequisites), then
run these commands from the repository root:

```sh
mise trust
mise install rust mr-boxington
mise run check
```

Use mise for both development and CI. Nix is not part of the toolchain.
`mise.toml` pins Rust 1.100.0 and a specific Mr Boxington release.
`Cargo.toml` declares the minimum supported Rust version, currently 1.85.

## Make a change

Keep each change focused on one problem. Put reusable behavior in `src/lib.rs`
and process I/O in `src/main.rs`. Add dependencies only when the change needs them.

Document public APIs with runnable examples and relevant error or panic
conditions. Add or update tests for changed behavior, bug regressions, or public
contracts. Prefer existing tests over duplicate coverage.

Use `#[cfg(test)] mod tests` for private internals, `tests/` for public API
integration tests, and rustdoc examples for usage. The starter's greeting example
is a doc test. Avoid tests that only check trivial wiring or implementation details.

## Run checks

Before submitting a code or build change, run:

```sh
mise run check
```

The task checks formatting, runs Clippy on all targets and features, runs tests
including doc tests, and builds documentation. Clippy and rustdoc warnings fail
the check. To fix formatting, run `mise exec -- cargo fmt --all`.

For compatibility changes, also test the declared MSRV:

```sh
mise exec -- rustup toolchain install 1.85.0 --profile minimal
mise exec -- cargo +1.85.0 test --locked
```

Both [CI jobs](.github/workflows/ci.yml) install tools with mise and run Cargo
through Mr Boxington. The **Binaries** workflow also packages macOS arm64/x64,
Linux x64, and Windows x64. Check platform-specific changes on the affected
platform and report what you tested.

## Manage dependencies and build storage

Use `mise exec -- cargo add <crate>` when adding a dependency. Include the
resulting `Cargo.toml` and `Cargo.lock` changes together. Keep `Cargo.lock` under
version control, including for library development. The `--locked` checks reject
unrecorded dependency changes.

mise's `mr_boxington = true` option wraps Cargo with mbx. Use `mise exec -- cargo
...` in non-interactive sessions, or activate mise in your shell. Editors should
inherit the same environment. Do not add `mbx setup` hooks or a `rustc-wrapper`.

mbx manages `target/` as a symlink and collects old build state automatically.
Inspect the cache with `mise exec -- mbx cache stats`. Clear this project's build
state with `mise exec -- mbx clean`. Do not use `cargo clean` or delete `target/`
directly. A remote compilation cache is not configured.

## Update tools and skills

Change tool selections in `mise.toml`, then run `mise install`
and `mise run check`. Keep Rust pinned to an exact release so local development
and CI use the same compiler, and update the documented version alongside it.

If the MSRV changes, update `Cargo.toml`, the CI job, and the documented commands
in README, AGENTS, and this file together. Test the new minimum toolchain.

Keep skill content in `.agents/skills/`. For a new skill, add a relative link in
`.claude/skills/`, following the existing links. Preserve the upstream license
and revision when updating vendored skills. The update procedure is recorded in
[`THIRD-PARTY-NOTICES.md`](THIRD-PARTY-NOTICES.md).

## Submit a pull request

Explain the problem, resulting behavior, and checks you ran. Include a reproducer
for a bug fix. Note any checks you could not run and any public API or MSRV changes.
Use small Conventional Commits, such as `fix: preserve empty input`.

Measure representative workloads before claiming a performance improvement.
crates.io publishing remains disabled by `publish = false`.

## Release workflow

Run `mise run check-release` after changing a workflow or packaging configuration.
It runs actionlint and validates dist's configuration. To inspect a local archive:

```sh
mise exec -- dist build --artifacts=local --target <your-host-triple>
```

The release tools are pinned in `mise.toml`; keep the dist version there and in
`dist-workspace.toml` aligned. `dist-workspace.toml` describes distribution only;
the project remains a single Cargo package. CI is maintained in `.github/workflows/`
to use mise and Mr Boxington throughout. dist's CI generation is disabled.
release-plz is scoped to `mise run release-plz -- <command>` so packaging runners
do not install the controller. Its pinned release has no Intel macOS binary.
The Intel macOS archive is cross-compiled on Apple Silicon because Mr Boxington
also has no Intel macOS release asset. The build installs the requested Rust target.

`release-plz.toml` enables git-only version detection and releases only after
merging a release PR. Its `[workspace]` section also applies to a single package.
Conventional Commit subjects determine version bumps and changelog categories;
review the generated release PR before merging. The first release is `0.1.0`.
Later releases compare against matching `v<version>` tags.
Library API comparison is disabled for this binary release flow; declare breaking
changes with `!` or a `BREAKING CHANGE:` footer in the Conventional Commit.

Enable **Allow GitHub Actions to create and approve pull requests** in the
repository's Actions settings. The read-only **CI** workflow must succeed on a
push to `main` before **Release-plz** runs. PR code never runs in that privileged
workflow. Bot-created release PRs receive an explicit **CI** dispatch because
`GITHUB_TOKEN` pushes do not trigger normal PR workflows. Stable tags likewise
receive an explicit **Binaries** dispatch. Only that workflow publishes releases,
after every platform build and checksum verification succeeds.
Main-branch CI runs are preserved so a later push cannot cancel the check that
triggers a release. Tagging runs separately from the serialized, rolling release
PR updates. Tagging uses the exact checked commit; PR preparation starts from
the latest `main`, and the resulting release PR receives its own checks.

If tagging succeeds but a binary build fails, rerun the failed **Binaries** run,
or dispatch it from `main` with the existing tag. The tag must name a commit
reachable from `main`, and its version must match `Cargo.toml`. If publication
left a draft, inspect and delete that incomplete draft before rerunning; keep
the tag. Never replace assets on an already-published stable release: fix the
problem and release a new version. If PR creation fails, check the repository
setting above, then rerun **Release-plz**.

After the first release, verify installation using the command in the README
on each supported platform. Before that release exists, local archive checks
and PR artifacts prove packaging; they do not prove hosted mise installation.
