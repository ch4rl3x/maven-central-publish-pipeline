# maven-central-publish-pipeline

Central GitHub Actions pipeline for the Kotlin Multiplatform libraries that are
published to Maven Central. Functionally identical to the workflows that used to
live in `settings-multiplatform`, but kept in one place instead of copied into
every repository.

The GitHub equivalent of GitLab's `include: project:` is a *reusable workflow*
(`on: workflow_call`). Unlike GitLab, no YAML is included textually — a reusable
workflow is called as a complete job. That is why every repository keeps a small
stub that does nothing but define the triggers.

## Usage

Copy `examples/snapshot.yml` and `examples/release.yml` into the consuming
repository's `.github/workflows/`, then delete the old workflows and its
`.github/workflows/actions/` directory. `mkver.conf` stays in the repository
root as before.

`secrets: inherit` is required — the workflows do not declare the six secrets
individually.

## Inputs

`snapshot.yml`:

| Input | Default | |
|---|---|---|
| `runner` | `macos-26-intel` | |
| `sample-tasks` | *(empty)* | e.g. `:sample-app:android-app:assemble`; empty skips the step |
| `run-tests` | `false` | test job (previously commented out) |

`release.yml`: `runner` only.

Everything else is hardcoded as before: JDK 17, `assembleRelease`,
`dokkaGeneratePublicationHtml`, `check`, git-mkver 1.3.0,
`publishAllPublicationsToSonatypeRepository`, `--max-workers 1`, snapshots from
`main`.

## Versioning

Consumers reference the movable major tag `@v1`. After a change, move the tag
forward and every repository picks the update up:

```bash
git tag -f v1 && git push -f origin v1
```

What follows `@` is a literal Git ref (tag, branch or full SHA), **not a version
range** — GitHub does not resolve `@v1` to "the latest v1.x". The convention
only works because the `v1` tag is moved by hand.

Cut a new major when inputs are renamed or removed. Bump the internal refs
first, then tag — otherwise the v2 workflow would load the v1 `prepare`:

```bash
sed -i '' 's|/actions/prepare@v1|/actions/prepare@v2|g' .github/workflows/*.yml
git commit -am "feat!: breaking change" && git tag v2 && git push origin main --tags
```

Then migrate the consumers from `@v1` to `@v2` one at a time; as long as `v1`
stays on the old commit, repositories that have not been migrated keep working.
Note that a movable major tag only covers one major at a time: once `main`
points at `@v2` internally, a v1 patch needs a `release/v1` branch.

The workflows reference `actions/prepare` fully qualified
(`ch4rl3x/maven-central-publish-pipeline/.github/actions/prepare@v1`) rather
than relatively, because inside a reusable workflow `uses: ./…` resolves against
the **calling** repository, not against this one.

While working on the pipeline itself, those internal references still point at
`@v1`. To test a branch, repoint them temporarily:

```bash
sed -i '' 's|/actions/prepare@v1|/actions/prepare@my-branch|g' .github/workflows/*.yml
```

## Differences from the previous setup

Only things that were broken or shut down:

- Removed `actions/cache@v3` (shut down) — it overlapped with `setup-java` and
  `cache: gradle`, which now handles caching on its own.
- `gradle/wrapper-validation-action@v2` (archived) → `gradle/actions/wrapper-validation@v4`.
- `concurrency.group` used `github.name`, which does not exist in the context and
  is always empty. Now `github.workflow`.
- `:sample-app:android-app:assemble` was hardcoded → `sample-tasks` input.
- The release tag name is passed through `env` instead of `${{ }}` inside the
  script body.

Not carried over: `.github/workflows/actions/doc`. It was commented out in both
workflows and uploaded its artifact as `compose-cache-dokka` — copy-paste from
the `compose-cache` repository.

## Later

- Runner split: only the Apple targets and publishing need macOS; the Android
  build and Dokka run on `ubuntu-latest`. At a 10x billing multiplier across 15
  repositories this is the biggest cost lever.
- A Renovate preset in its own repository for all repositories to extend.
- Extract the convention plugins into `ch4rl3x/gradle-conventions` — independent
  of this pipeline, which only ever calls `./gradlew <task>`.
