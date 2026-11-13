# Binary releases and changelogs

Researched 2026-11-13 against project documentation, source, and live GitHub
repository metadata. This is a recommendation; no release automation is enabled.

## Recommendation

Start with **release-plz for versions, changelogs, and release PRs**, with
**dist (cargo-dist) for platform binaries and GitHub Release assets**. This is
the simpler Rust-specific default for this single-package starter. Conventional
Commits select version bumps; release-plz embeds git-cliff for changelog
generation, so no separate git-cliff installation or workflow is needed.
The release PR provides a place to review and edit the resulting notes.
[release-plz](https://github.com/release-plz/release-plz),
[changelog implementation](https://release-plz.dev/docs/changelog),
[release PRs](https://release-plz.dev/docs/github/quickstart).

Choose **Knope + dist** if Changesets means an authored Markdown change file in
each PR. Contributors put a release type and description in `.changeset/`;
Knope combines the files into `CHANGELOG.md` and updates package versions. Its
format explicitly follows JavaScript Changesets, supports Cargo manifests and
lockfiles, and is used in Knope's own repository.
[ChangeSet documentation](https://knope.tech/reference/concepts/changeset/),
[release preparation](https://knope.tech/reference/config-file/steps/prepare-release/),
[Knope's configuration](https://github.com/knope-dev/knope/blob/main/knope.toml).

Knope is a smaller project than release-plz or release-please; all three were
active and unarchived when checked. Neither option is a universal Rust standard.
[Knope](https://github.com/knope-dev/knope),
[release-plz](https://github.com/release-plz/release-plz),
[release-please](https://github.com/googleapis/release-please).

## Options considered

| Tool | Versions and changelogs | Fit here |
| --- | --- | --- |
| [Knope](https://knope.tech/) | Authored `.changeset/*.md` files and optional Conventional Commits; release PRs and GitHub Releases | Closest match to Changesets; keep a small explicit configuration. |
| [release-plz](https://release-plz.dev/docs/config#the-git_only-field) | Rust-aware release PRs, Conventional Commits, embedded git-cliff | Strong default if commit-derived notes are acceptable. `git_only = true` supports this package's `publish = false`. |
| [release-please](https://github.com/googleapis/release-please#strategy-language-types-supported) | Conventional Commits, release PRs, Rust manifest/lockfile updates | Established cross-language option; binary building is separate, and it does not provide the requested change-file workflow. |
| [cargo-release](https://github.com/crate-ci/cargo-release) | Cargo versioning, tagging, publishing, hooks | Good maintainer-driven release command; more assembly is needed for release PRs, changelog collection, and binary delivery. |

Use one version/changelog tool. dist handles a different job: generating CI,
building archives, checksums, and installers, then attaching them to a release.
Astral's uv uses dist, demonstrating substantial production use; its extensive
custom build configuration should not be copied into this starter.
[dist](https://github.com/axodotdev/cargo-dist),
[uv configuration](https://github.com/astral-sh/uv/blob/main/dist-workspace.toml).
The active upstream is `axodotdev/cargo-dist`; the `astral-sh/cargo-dist` fork
was archived when checked.

## What runs on each event

| Event | Recommended result |
| --- | --- |
| Pull request push | Run checks and build downloadable preview artifacts. dist supports `pr-run-mode = "upload"`; these are Actions artifacts, not mise-installable releases. |
| Push to `main` | Update one release PR containing the next version and accumulated changelog. Optionally publish an immutable snapshot prerelease for each successful commit. |
| Release PR merge | Tag the merged commit, build all supported binaries, and publish the stable GitHub Release after successful builds. |

release-plz and Knope document the release-PR pattern; dist documents PR builds.
An every-commit snapshot publisher is additional workflow logic, rather than
something this combination enables automatically. Use unique identities such as
`0.2.0-dev.<run-number>` and record the source SHA. Keep stable versions for
reviewed releases. If a stable release on every change is desired, the release
PR can later be automatically merged once its checks pass.
[release-plz release gating](https://release-plz.dev/docs/config#the-release_always-field),
[Knope release PR recipe](https://github.com/knope-dev/knope/blob/main/docs/src/content/docs/recipes/1-preview-releases-with-pull-requests.md),
[dist PR builds](https://axodotdev.github.io/cargo-dist/book/reference/config.html#pr-run-mode).

## Installing with mise

GitHub Release assets are sufficient; no crates.io publication or custom mise
plugin is required. Once the first release exists:

```sh
mise use -g github:jkomyno/rust-template@0.1.0
mise exec -- rust-template
```

For consumers who want the newest release, including snapshots:

```toml
[tools]
"github:jkomyno/rust-template" = { version = "latest", prerelease = true }
```

The GitHub backend selects a platform asset and extracts it. It excludes
prereleases from normal version discovery by default and never lists drafts;
`prerelease = true` opts into discovery including prereleases. A published
snapshot prerelease can therefore be installed; an Actions artifact cannot be
installed through this backend. This setting includes stable releases too; pin a
snapshot version when a specific build is required. The commands describe the
proposed future release layout; installation against this repository still needs
verification after the first release.
[mise GitHub backend](https://mise.jdx.dev/dev-tools/backends/github.html).

## Implementation outline

1. Pin release-plz and dist in mise. Use Conventional Commit PR titles with squash
   merges. Configure `git_only = true` and `release_always = false` in
   `release-plz.toml`: track versions from tags and release after merging the
   release PR. Maintain `Cargo.toml`, `Cargo.lock`, and `CHANGELOG.md` in that PR.
   The `[workspace]` section in release-plz's configuration also applies to a
   single Cargo package; no Cargo workspace is needed.
   [release-plz configuration](https://release-plz.dev/docs/config).
2. Keep `publish = false`: this disables crates.io publishing. Explicitly opt
   the binary into dist with `dist = true`, because dist normally treats
   `publish = false` as a reason to skip distribution. No Cargo registry token
   is needed for this design. [dist package selection](https://axodotdev.github.io/cargo-dist/book/reference/config.html#dist).
3. Start with macOS arm64/x64, Linux x64, and Windows x64 archives and
   checksums. Run builds through mise and Mr Boxington as required here;
   review generated dist CI rather than accepting a second toolchain setup.
4. Give dist ownership of the GitHub Release. Set release-plz's
   `git_release_enable = false` while keeping tags enabled; let dist create the
   release and use `CHANGELOG.md` for its notes. If using Knope-created draft
   releases instead, dist supports `create-release = false` to upload to and
   publish an existing draft. Coordinate creation before starting the binary
   workflow. [release-plz release settings](https://release-plz.dev/docs/config#the-git_release_enable-field),
   [dist release ownership](https://axodotdev.github.io/cargo-dist/book/reference/config.html#create-release).
5. Use a narrowly scoped GitHub App token when the release tool's tag push must
   start dist's separate workflow. Tags/PRs created with the default
   `GITHUB_TOKEN` do not start normal push/PR workflows. An alternative is an
   explicit workflow dispatch or a single workflow that passes outputs to its
   build jobs. [Token and trigger behavior](https://release-plz.dev/docs/github/token),
   [dist dispatch support](https://axodotdev.github.io/cargo-dist/book/reference/config.html#dispatch-releases).
6. Validate a draft release and a real mise installation on each supported OS
   before enabling stable publication. Add per-commit snapshots only if they
   are needed; publish from trusted branch code, while PR builds remain
   unprivileged artifact uploads.
