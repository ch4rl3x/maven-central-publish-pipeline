# maven-central-publish-pipeline

Shared GitHub Actions workflows for the libraries published to Maven Central, so
the CI lives in one place instead of being copied into every repository.

GitHub's equivalent of GitLab's `include: project:` is a *reusable workflow*
(`on: workflow_call`). It is called as a complete job, not included as text, so
every repository keeps a small stub that only defines the triggers.

## Workflows

| Workflow | For | Runner |
|---|---|---|
| `kmp-snapshot.yml` | Kotlin Multiplatform libraries | `macos-26-intel` |
| `jvm-snapshot.yml` | JVM libraries, Gradle plugins, Android libraries | `ubuntu-latest` |
| `release.yml` | both, on a GitHub release | `macos-26-intel` |

The snapshot workflows build, optionally test, and publish a SNAPSHOT from `main`.
`release.yml` publishes the release tag and closes the Sonatype staging repository.

There is no `kmp-release.yml` / `jvm-release.yml`: the release path does not depend
on the project shape, only on the runner, and that is an input.

## Usage

Copy the matching files from `examples/` into the consuming repository's
`.github/workflows/`. `mkver.conf` stays in the repository root.

```yaml
jobs:
  snapshot:
    uses: ch4rl3x/maven-central-publish-pipeline/.github/workflows/kmp-snapshot.yml@v1
    secrets: inherit
```

`secrets: inherit` is required -- the workflows do not declare the six secrets
(`OSSRH_USERNAME`, `OSSRH_PASSWORD`, `SONATYPE_STAGING_PROFILE_ID`,
`SIGNING_KEY_ID`, `SIGNING_KEY`, `SIGNING_KEY_PASSWORD`) individually. They have
to be repository secrets; environment secrets are only visible to a job that
declares an `environment`, and these do not.

GitHub shows the jobs of a reusable workflow as `<calling job> / <called job>`,
which is why the calling job is named `snapshot` or `release`: the checks then read
`snapshot / build`, `snapshot / publish`, `release / publish`.

## Inputs

`kmp-snapshot.yml`:

| Input | Default | |
|---|---|---|
| `runner` | `macos-26-intel` | |
| `sample-tasks` | *(empty)* | e.g. `:sample-app:android-app:assemble`; empty skips the step |
| `run-tests` | `false` | runs `check` and reports the results |

`jvm-snapshot.yml`:

| Input | Default | |
|---|---|---|
| `runner` | `ubuntu-latest` | |
| `run-tests` | `true` | runs `check` and reports the results |

`release.yml`: `runner` only -- pass `ubuntu-latest` for JVM and Android projects.

Everything else is fixed: JDK 17, `assembleRelease` (KMP) / `assemble` (JVM),
`dokkaGeneratePublicationHtml`, `check`, git-mkver 1.3.0,
`publishAllPublicationsToSonatypeRepository`, `--max-workers 1`, snapshots from
`main`.

Tests run as steps inside the `build` job rather than as a job of their own, so
that `publish` -- which waits on `build` -- is gated by them. The test report is
skipped when a repository produces no JUnit results.

## Versioning

Consumers reference the movable major tag `@v1`. After a change, move it forward
and every repository picks the update up:

```bash
git tag -f v1 && git push -f origin v1
```

What follows `@` is a literal Git ref, **not a version range** -- GitHub does not
resolve `@v1` to "the latest v1.x".

The workflows reference `actions/prepare` fully qualified
(`ch4rl3x/maven-central-publish-pipeline/.github/actions/prepare@v1`) rather than
relatively, because inside a reusable workflow `uses: ./…` resolves against the
**calling** repository. For a new major, bump those internal refs first, then tag:

```bash
sed -i '' 's|/actions/prepare@v1|/actions/prepare@v2|g' .github/workflows/*.yml
```

Then migrate consumers one at a time; as long as `v1` stays on the old commit,
repositories that have not moved keep working.

## Open

- The KMP workflows run on macOS. A library only publishes `.klib` files for its
  Apple targets, and Kotlin/Native can produce those on any host -- only final
  binaries and cinterop need macOS. If that holds, `kmp-snapshot.yml` and
  `release.yml` could default to `ubuntu-latest`, which is the largest cost item
  across all repositories.
- A Renovate preset in its own repository for every repository to extend.
