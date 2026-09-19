# Releasing

How every repository in this organisation cuts a release. Written for people and for
coding agents alike; a repository's own `CONTRIBUTING.md` or `AGENTS.md` should point here
rather than restate it.

The short version: **releases are cut from `main` by pushing a `vX.Y.Z` tag.** Everything
before the tag is one ordinary commit; everything after it is automated.

## Before the tag

1. **Write the changelog as changes land**, not at release time. `CHANGELOG.md` follows
   [Keep a Changelog](https://keepachangelog.com/en/1.1.0/): new entries go under
   `## [Unreleased]`, and each says *why*, not just what. Dependency bumps get an entry
   too, stating what the upgrade needed. The release tooling moves the heading; it never
   writes the content, so an empty `[Unreleased]` becomes a release with empty notes.
2. **Decide the version.** This is a judgement call and is deliberately not automated.
   Pre-1.0, a breaking change bumps the minor. Breaking includes things no tool can
   infer from a commit subject: a raised MSRV, a changed on-disk or wire format, a
   removed feature flag.
3. **One release commit**, subject `chore: release vX.Y.Z`, touching only the version
   field of the manifest and `CHANGELOG.md` — rename `[Unreleased]` to `[X.Y.Z] - date`,
   open a fresh empty `[Unreleased]` above it, and update the link definitions at the
   bottom.
4. **Tag and push.** `git tag -a vX.Y.Z -m vX.Y.Z && git push origin main vX.Y.Z`. Push
   the branch before or with the tag, never after: the release workflow checks out the
   tag independently, but notes whose `[Unreleased]` compare link points at a tag the
   branch has not reached yet 404 until it catches up.

### Rust

`cargo release` does steps 3 and 4 in one command, driven by a `release.toml` in the
repository:

```bash
cargo release 0.3.0            # dry run: prints every edit, changes nothing
cargo release 0.3.0 --execute
```

Pull `main` first; rebase and squash merges rewrite commits, and `cargo release` refuses
to run from a branch that has diverged. The reference `release.toml` is
[`templates/release.toml`](templates/release.toml). It sets `publish = false`.

## After the tag

Pushing the tag triggers the repository's `.github/workflows/release.yml`, which should
call the shared workflow rather than carry its own copy:

```yaml
name: Release
on:
  push:
    tags: ["v*"]
permissions:
  contents: write
  id-token: write      # only needed when publishing
jobs:
  release:
    uses: anaxo-io/.github/.github/workflows/release-rust.yml@main
    with:
      publish: false   # true once the crate exists on crates.io; see below
```

The shared workflow:

1. **refuses the tag if it does not match the manifest version** — the release action
   reads the changelog but never the manifest, so nothing else catches this;
2. **runs the artefact check** — for Rust, `cargo package`, which builds the crate from
   exactly the files that would ship (repositories that cannot package, for instance
   because of a git dependency, pass `package: false` and get `cargo check --all-features`
   instead);
3. **creates the GitHub release** with
   [`taiki-e/create-gh-release-action`](https://github.com/taiki-e/create-gh-release-action),
   taking the notes from the matching `CHANGELOG.md` section and failing if there is
   none, so the release and the changelog cannot disagree. A tag containing a hyphen
   (`v0.3.0-rc1`) is published as a pre-release.

It deliberately does **not** re-run the test suite. Releases come from `main`, every
commit there has passed CI, and this workflow runs once per release with a cold build
cache: repeating the suite costs ten minutes to learn what CI reported minutes earlier.

Repositories that ship executables add a build-and-upload matrix on top; see
[`hmedkouri/cc-ledger`](https://github.com/hmedkouri/cc-ledger/blob/main/.github/workflows/release.yml)
for the shape.

### Publishing to crates.io

Off by default. A crate that publishes passes `publish: true` to the shared workflow and
grants `id-token: write` in its caller; the workflow then publishes after the GitHub
release using [Trusted Publishing](https://crates.io/docs/trusted-publishing) — GitHub
proves the workflow's identity per run and no long-lived token exists anywhere. That is
the only place publishing happens; `release.toml` keeps `publish = false`, because a
published version can be yanked but never replaced or reused, and a command anyone can
run from a laptop is the wrong place for that.

Getting there is two one-off steps, because Trusted Publishing needs the crate to exist:

1. **First version by hand.** Make the repository public (a crate whose `repository`
   link 404s looks abandoned), then from a worktree at the tag — not from `main` —
   `cargo publish --dry-run`, then `cargo publish` with a crates.io token that is
   short-lived, scoped to `publish-new` only and to that one crate. Revoke it after.
2. **Register the workflow.** On the crate's crates.io settings page, add the trusted
   publisher: this organisation, the repository, workflow file `release.yml`. Then set
   `publish: true` in the caller.

`X.Y.Z` on crates.io is the tag `vX.Y.Z`, byte for byte.

## Branch rules that go with this

`main` is protected the same way everywhere:

- pull request required, **zero approvals** — a single maintainer cannot approve their
  own pull request, so requiring one locks the repository;
- all CI checks required, branch up to date first;
- linear history; merge commits disabled; prefer **rebase** so every commit on `main`
  builds alone, squash only for `wip`/fixup noise;
- conversations resolved before merge; merged branches auto-delete;
- force pushes and deletion blocked;
- `enforce_admins` **off**, so `cargo release` can push its release commit directly.

**Renaming a CI job requires updating the required-checks list in the same change.** A
required check that no longer reports blocks every pull request forever, waiting for a
result that cannot arrive.

## Things that will surprise you

- **A tag event uses the workflow file from the tagged commit, not from `main`.** Fixing
  the workflow does nothing for tags that already exist. If a release run fails for a
  reason fixed after the tag, create the release by hand:
  `gh release create vX.Y.Z --notes-file <(parse-changelog CHANGELOG.md X.Y.Z)`.
- **Skipping CI with `paths-ignore` deadlocks pull requests.** A workflow skipped by a
  path filter never reports its checks, and required checks that never report block the
  merge forever. Gate jobs with `if:` instead; a skipped job still reports.
- **Retroactive releases do not happen.** A tag pushed before the release workflow
  existed has no release; create it by hand as above.
