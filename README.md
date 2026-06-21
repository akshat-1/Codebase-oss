# Build-and-Promote Pipeline

This `.github` folder gives a repo an automated pipeline: when someone opens
or updates a pull request against a designated branch, it builds the code
on a self-hosted runner using the right language toolchain, and — if the
build succeeds — automatically opens a second pull request carrying the
same changes against a release branch.

This document is the authoritative reference for how the whole thing works.
Comments inside the workflow/action files may be trimmed or absent; nothing
here depends on them.

---

## 1. The mental model

Three branches are involved, conceptually:

- **Trigger branch** (commonly `main` or `test`) — where people actually
  open PRs from their feature branches. This is the branch this pipeline
  watches.
- **Release branch** (commonly `release`) — the destination. Only changes
  that have passed an automated build ever get a PR opened against this
  branch.
- **Feature/working branches** — where individual changes are authored,
  PR'd against the trigger branch.

The pipeline's job: watch PRs targeting the trigger branch, build them, and
if the build passes, open an equivalent PR against the release branch.

### This is a pre-merge check, not a post-merge promotion

An important, deliberate design choice: **the build happens as soon as the
PR is opened (or updated) — it does not wait for the PR to be merged into
the trigger branch.**

Concretely:
1. Someone opens a PR: `feature/my-change` -> `trigger_branch`.
2. The workflow fires immediately (on `opened`, `synchronize`, or
   `reopened`).
3. It checks out the trigger branch, merges the PR's branch into it
   **locally, in the runner's own throwaway workspace** — this merge is
   never pushed anywhere, it exists only so the build sees what the code
   would look like *if* the PR were merged.
4. It installs the right toolchain and runs the build against that local
   merge result.
5. If the build succeeds, it takes `feature/my-change` (the PR's actual
   branch, not the throwaway local merge) and opens a **separate** PR:
   `feature/my-change` -> `release_branch`.

This means the release-branch PR can exist and be visible for review
**before** the original PR has been approved or merged into the trigger
branch. Two independent reviews end up happening on essentially the same
diff — one against the trigger branch (the normal review), one against
release (gated by the fact that it only appears once a build passed). This
is intentional, not a bug: it surfaces "this is mergeable and builds
cleanly" as early as possible, rather than waiting for the full review
cycle on the trigger branch to finish first.

If new commits are pushed to the same PR, the workflow re-runs and the
release-branch PR is updated in place (same branch, force-pushed) rather
than creating duplicates.

### Why the local merge step exists at all

Building only the PR's branch in isolation wouldn't tell you whether it's
compatible with the *current* state of the trigger branch — someone else's
already-merged change might conflict with this PR even though the PR looks
fine on its own. The local merge step catches that: it's the same kind of
check GitHub's own "this branch is out of date" indicator does, except it
actually runs the build against the merged result, not just checks for
textual conflicts.

If this local merge fails (a genuine merge conflict), the workflow stops
immediately, comments on the PR explaining why, and never reaches the build
step at all.

---

## 2. Trigger and branch matching

The workflow listens to `pull_request` events (`opened`, `synchronize`,
`reopened`) on **any** branch — the `branches:` filter is intentionally
left wide open (`'**'`). The actual filtering happens inside the workflow
itself: the very first job (`resolve-config`) compares the PR's **base**
branch against the configured trigger branch, and sets a `should_run` flag.
The second job (`build-and-promote`) only executes if that flag is `true`.

This two-job split exists so that configuration resolution (figuring out
*what* the trigger/release branches and build settings even are) happens
before anything expensive or destructive runs, and so a PR targeting an
unrelated branch is skipped cleanly with a clear log line rather than
silently doing nothing or erroring.

**A critical, non-obvious requirement:** because this is a `pull_request`
workflow, GitHub determines which version of the `.github/workflows/*.yml`
file to run based on what exists **on the PR's base branch** — not
whatever's on `main` or wherever the "canonical" copy lives. If a PR
targets a branch (e.g. a `test` branch) that doesn't have its own copy of
this `.github` folder, **GitHub won't find or run the workflow at all** —
not even to skip it, it simply won't trigger. If you intend to use more
than one trigger branch over time, or you're testing on a non-`main`
branch, **make sure that branch also has its own copy of `.github`**, kept
in sync with whichever branch is the "real" source of truth for this
pipeline. There's no automation for this sync in this pipeline — it's a
manual responsibility.

---

## 3. Configuration — no YAML editing required

Settings are resolved in this priority order, for every setting below:

**Repo variable -> `.github/promote-config.yml` -> ecosystem-based default
(where one exists) -> hardcoded fallback.**

### Where to set repo variables

GitHub repo -> **Settings -> Secrets and variables -> Actions -> Variables
tab -> New repository variable.**

### Settings

| Setting | Repo variable | Config file key | Default if unset |
|---|---|---|---|
| Trigger branch | `TRIGGER_BRANCH` | `trigger_branch` | `main` |
| Release branch | `RELEASE_BRANCH` | `release_branch` | `release` |
| Install command | `INSTALL_COMMAND` | `install_command` | ecosystem default (see below) |
| Build command | `BUILD_COMMAND` | `build_command` | ecosystem default (see below) |
| Node version | `NODE_VERSION` | `node_version` | auto-detected, else `20.18.1` |
| Python version | `PYTHON_VERSION` | `python_version` | `3.12.7` |
| Java version | `JAVA_VERSION` | `java_version` | auto-detected, else `21` |
| Maven version | `MAVEN_VERSION` | `maven_version` | `3.9.9` |
| Gradle version | `GRADLE_VERSION` | `gradle_version` | `8.10.2` |
| Go version | `GO_VERSION` | `go_version` | auto-detected, else `1.22.9` |
| Rust version | `RUST_VERSION` | `rust_version` | `stable` |

`.github/promote-config.yml` is read directly from the repo (checked into
git, editable like any file) and is only consulted for a setting if the
matching repo variable isn't set.

If `BUILD_COMMAND` resolves to empty (no variable, no config file value,
and the detected ecosystem has no built-in default), the workflow fails
immediately with a clear error rather than silently doing nothing.

---

## 4. Ecosystem detection

A dedicated composite action, `.github/actions/detect-ecosystem`, inspects
the repo's root for specific marker files and returns one of: `node`,
`maven`, `gradle`, `python`, `go`, or `unknown`. Detection order (first
match wins):

1. `package.json` -> `node`
2. `pom.xml` -> `maven`
3. `build.gradle` or `build.gradle.kts` -> `gradle`
4. `requirements.txt`, `pyproject.toml`, or `setup.py` -> `python`
5. `go.mod` -> `go`
6. none of the above -> `unknown`

Note Maven and Gradle are reported as **separate** ecosystem values, not a
combined "java" — this is deliberate, since they need different build
tools installed (`mvn` vs `gradle`) even though both run on a JDK.

There is currently no detection rule for Rust (no `Cargo.toml` check) — a
Rust repo will be classified `unknown` and the workflow will fail with a
clear message rather than silently mis-building, unless `BUILD_COMMAND` is
set explicitly to override detection entirely.

If you need a different/more sophisticated detector, replace the logic
inside `detect-ecosystem/action.yml` — the only contract the rest of the
pipeline relies on is that it sets an output named `ecosystem`.

### Ecosystem to command/toolchain mapping

| Detected ecosystem | Default install command | Default build command | toolchain_id passed to setup-toolchain |
|---|---|---|---|
| `node` | `npm ci` | `npm run build` | `node` |
| `python` | `pip install -r requirements.txt` | `python -m build` | `python` |
| `maven` | (none) | `mvn -B package` | `java-maven` |
| `gradle` | (none) | `gradle build` | `java-gradle` |
| `go` | `go mod download` | `go build ./...` | `go` |
| `rust` | (none — not auto-detected, see above) | `cargo build --release` | `rust` |

---

## 5. Version auto-detection

For **Java, Node, and Go specifically**, if no explicit version is set via
repo variable or config file, the workflow tries to read the actual
intended version straight out of the codebase before falling back to a
hardcoded default. This avoids drift between "what the project declares it
needs" and "what version the CI pipeline happens to install."

| Language | Source(s) checked, in order | Pattern(s) matched |
|---|---|---|
| Java | `pom.xml` | `<maven.compiler.source>`/`<maven.compiler.target>`, then `<java.version>`, then `<release>` |
| Java | `build.gradle` / `build.gradle.kts` (if no pom.xml match) | `sourceCompatibility = '...'`, then `JavaLanguageVersion.of(N)` |
| Node | `package.json` | `"engines": { "node": "..." }` — first number found, handles ranges like `>=18.0.0 <21`, `^18`, `18.x`, etc. |
| Go | `go.mod` | the `go 1.22` directive line |

Java version normalization: legacy `1.8`-style notation is converted to
`8`, and any minor/patch suffix is stripped (`17.0.2` becomes `17`), since
the JDK installer in `setup-toolchain` expects a bare major version number.

All pattern matching uses plain POSIX `grep -E` (no PCRE `-P` flag) — this
was a deliberate choice because `grep -P` support isn't guaranteed on
minimal Linux installs (confirmed this matters on at least one of the
runners this pipeline has been tested against). If a file doesn't match
any known pattern, detection silently yields nothing and the hardcoded
default is used — this never causes a failure on its own.

Python, Maven, Gradle, and Rust tool versions are **not** auto-detected;
they always come from the repo variable -> config file -> hardcoded
default chain only.

---

## 6. How builds actually run — no Docker

Toolchains are installed **directly on the self-hosted runner**, not in
Docker containers. This was a deliberate pivot partway through building
this pipeline: the runner(s) this has actually been tested against had
Docker available on one machine but not consistently across all of them,
so the design settled on direct installation as the approach that works
everywhere without assuming Docker is present.

`.github/actions/setup-toolchain` is a composite action that installs
whichever toolchain the detected ecosystem needs:

| Ecosystem | What gets installed | Method |
|---|---|---|
| `node` | Node.js | Prebuilt tarball |
| `java-maven` | Temurin JDK + Apache Maven | Two prebuilt tarballs |
| `java-gradle` | Temurin JDK + Gradle | Prebuilt tarball + zip |
| `go` | Go | Prebuilt tarball |
| `rust` | Rust (via rustup) | rustup shell installer, scoped to a version-specific install dir |
| `python` | Python, built from source | Source tarball + system compiler/dev packages, then `./configure && make altinstall` |

### Caching

Every install is cached under `$HOME/.toolchains/<name>-<version>` — e.g.
`$HOME/.toolchains/temurin-21`, `$HOME/.toolchains/gradle-8.10.2`. Before
installing anything, the relevant step checks whether that exact
version's binary already exists at that path; if so, the download/install
is skipped entirely. This makes repeat runs fast except for the very first
time a given version is needed on a given runner. Nothing is automatically
cleaned up — disk usage grows over time as new versions accumulate (see
Known Limitations).

### Why $HOME, not /opt or a system-wide location

Earlier in this pipeline's development, toolchains were installed under
`/opt/toolchains`, which requires root or sudo to write to. This broke on
a runner that operates as a **non-root** user account, where `sudo` itself
was also found to be unreliable (hung indefinitely even on a simple
non-interactive check). Moving everything under `$HOME/.toolchains`
sidesteps the problem entirely: the runner's own account always owns its
home directory, so no elevated privileges are ever needed for Node, Java,
Maven, Gradle, Go, or Rust installation, regardless of which account the
runner process runs as or whether `sudo` works on that machine.

**Python is the one exception.** Building Python from source requires
system-level compiler and development-library packages (`gcc`, `make`,
various `-devel`/`-dev` packages) installed via the OS package manager
(`dnf`/`yum`, `apt-get`, or `pacman` — auto-detected). Installing system
packages genuinely requires root. If the runner account is not root and
`sudo` isn't reliably available, **Python builds will fail** at the
package-install step. This is a known, currently-unresolved gap — see
Known Limitations.

`setup-toolchain` deliberately makes **no `sudo` calls anywhere** — every
privileged command (the Python system-package installs) runs directly,
under the assumption that if root access is needed, the runner process
itself is already running as root. This was changed partway through
development after discovering that calling `sudo` non-interactively from
within a GitHub Actions step has no way to supply a password and will hang
or fail outright — there is no safe way to embed a password in the
workflow (it would be committed to git history permanently), so the only
supported paths are: run the runner as root, or pre-install Python's
system build dependencies manually once.

---

## 7. The gh CLI

The pipeline uses the GitHub CLI (`gh`) to open and update pull requests
and post comments. Rather than assuming it's pre-installed, the
`build-and-promote` job includes an **"Ensure gh CLI is available"** step
that runs before anything else: it checks `command -v gh`, and if missing,
downloads the official `gh` release tarball directly (no package manager,
no root needed) into `$HOME/.toolchains/gh-<version>`, then adds it to
`PATH` for the rest of the job via `$GITHUB_PATH`. Architecture (amd64 vs
arm64) is detected automatically via `uname -m`; an unsupported
architecture fails with a clear error rather than attempting a download
that would fail anyway.

`gh` authenticates automatically via the `GH_TOKEN` environment variable
set on the job (see Authentication below) — no separate `gh auth login`
step or stored credentials are needed.

---

## 8. Authentication and permissions

### GITHUB_TOKEN vs REPO_PAT

Every git push and every `gh` call in this pipeline uses:

    secrets.REPO_PAT || secrets.GITHUB_TOKEN

meaning: use a custom PAT secret named `REPO_PAT` if one is set, otherwise
fall back to the default token GitHub provides automatically to every
workflow run.

**This matters because the default `GITHUB_TOKEN` may not have permission
to push branches or open PRs**, depending on the repo's settings under
**Settings -> Actions -> General -> Workflow permissions**. If that's set
to "Read repository contents permission" (the safer, more restrictive
GitHub default), `GITHUB_TOKEN` is read-only and every push in this
pipeline will fail with a `403 Permission denied` error. Two fixes:

- **Simpler:** switch that setting to "Read and write permissions". This
  affects every workflow in the repo, not just this one.
- **More contained:** leave the repo-wide setting as-is, and instead
  create a fine-grained Personal Access Token scoped to just this repo
  (Contents: read/write, Pull requests: read/write), store it as a repo
  secret named exactly `REPO_PAT`. The pipeline will automatically prefer
  it over the default token.

### Branch protection on the release branch

If the release branch has protection rules (required reviews, restricted
pushers, etc.), confirm whichever token is actually in use (`GITHUB_TOKEN`
or `REPO_PAT`) has permission to push branches and open PRs against it —
otherwise the promotion step will fail even with the workflow-permissions
setting above correctly configured.

---

## 9. Private Docker registry support (currently unused, but built in)

Earlier in this pipeline's design, builds ran inside Docker containers
pulled from a private registry rather than Docker Hub — this is no longer
how builds run (see section 6), but the registry-authentication mechanism
that was built for it is worth documenting in case Docker is reintroduced
later, or if any remaining Docker-based step still references it.

If reintroduced: `REGISTRY` (repo variable or config key) is prepended to
whatever Docker image is resolved, unless the image already contains a
`/`. `REGISTRY_USERNAME` and `REGISTRY_PASSWORD` are **secrets only, never
variables or config file values** — `docker login` would run once near the
start of the job using `--password-stdin` (so the credential never appears
in process args or logs), and `docker logout` would run unconditionally at
the end of the job, since a long-lived self-hosted runner persists Docker
credentials between unrelated runs otherwise.

---

## 10. Fetching toolchain binaries from a private Artifactory mirror

Some organizations require dependency/binary downloads to come from an
internal Artifactory instance rather than the public internet, even when
the runner has confirmed network access to public sources. Investigation
into this for the current setup found:

- The runner's network position grants **unauthenticated** access to at
  least one confirmed internal Artifactory repo (`node-dist`) — a plain
  `curl` with no auth header, run from the runner itself, returns a real
  `200` and the actual file content, not a login page. This means no
  token or credential handling was needed for this particular repo.
- `node-dist` is a straight mirror of the public `nodejs.org/dist/`
  layout — same path structure, just a different base URL.

### Current state: placeholders, not fully wired up

`.github/actions/setup-toolchain/action.yml` has been updated so that:

- **Node.js** downloads from the confirmed, working Artifactory URL
  (`.../artifactory/node-dist/v<version>/node-v<version>-linux-x64.tar.gz`)
  — this is live now, not a placeholder.
- **Java/Temurin, Maven, Gradle, and Go** each have a clearly-named
  placeholder variable (e.g. `ARTIFACTORY_JAVA_URL`) set to a dummy URL
  containing `REPLACE_ME_<TOOL>_REPO` and `REPLACE_ME_PATH`, sitting
  alongside the still-active public-source download. To find every
  remaining placeholder, search the file for the text `REPLACE_ME`.
  Filling one in means: find the real Artifactory path for that tool,
  replace the placeholder URL's value, and swap which `curl` line is
  actually used (comment out or remove the public-source one).
- **Rust** was intentionally left without a same-shaped placeholder. The
  public Rust install method (`rustup`) is a shell installer script, not a
  static tarball — if Artifactory mirrors Rust at all, it likely needs a
  different integration approach (either a mirrored copy of the
  `rustup-init.sh` script, or pre-built toolchain tarballs as a totally
  separate install method from rustup). This needs a decision once/if a
  Rust mirror is found, not just a URL swap.
- **Python** (source tarball from python.org) and **gh CLI** (from GitHub
  releases) have not been investigated for Artifactory mirrors at all and
  still pull from their original public sources unconditionally.

No authentication scaffolding (tokens, secrets) has been added for
Artifactory yet, since the one path tested so far didn't need any. If a
different Artifactory repo turns out to require a token, the pattern to
follow is the same one used for the private Docker registry (section 9):
a token value stored as a repo secret, never a variable or config file
value, passed via a curl Authorization header.

---

## 11. Known limitations

- **Merge conflicts** between the PR's branch and the trigger branch (the
  local-merge check) or between the PR's branch and the current release
  branch (the promotion step) are not auto-resolved. Either failure stops
  the pipeline and posts an explanatory comment on the PR; a human needs
  to resolve the conflict manually.
- **Python builds require root** for system package installation (`gcc`,
  `make`, dev libraries). On a non-root runner without reliable `sudo`,
  Python builds will fail at that step until either those packages are
  pre-installed manually, or a different mechanism is added.
- **No Rust detection** in `detect-ecosystem` — a Rust project will be
  classified `unknown` and fail with a clear error unless `BUILD_COMMAND`
  is set explicitly.
- **`$HOME/.toolchains` grows unboundedly** — old versions are never
  automatically cleaned up. Periodically prune manually if disk space
  becomes a concern.
- **Self-hosted runners are not ephemeral** by default. `clean: true` is
  set on checkout, but if multiple repos share the same runner machine,
  confirm the runner's own workspace-wiping behavior between jobs matches
  your expectations.
- **No automated sync between branches' copies of `.github`.** As noted in
  section 2, every branch this pipeline triggers on needs its own copy of
  these files, and there is currently no mechanism keeping them in sync
  automatically — edits made to one branch's copy must be manually
  propagated to the others.
- **Different repos/PRs requesting different exact toolchain versions is
  supported** (each version gets isolated under its own
  `$HOME/.toolchains/<name>-<version>` directory) but **concurrent jobs
  installing the same missing version at the same time could race** — not
  handled by this pipeline.
- **Artifactory integration is partial** — see section 10. Most toolchains
  still pull from the public internet; only Node has been fully switched
  over and confirmed working.

---

## 12. File reference

    .github/
    |-- README.md                          - this file
    |-- promote-config.yml                 - fallback configuration (no YAML editing needed)
    |-- actions/
    |   |-- detect-ecosystem/action.yml    - inspects repo root, outputs an ecosystem value
    |   `-- setup-toolchain/action.yml     - installs the toolchain matching a toolchain_id input
    `-- workflows/
        `-- promote-to-release.yml         - the orchestrator: trigger, config resolution, build, promotion

Any other files under `.github/workflows/` with names like `diagnose-*.yml`
or `check-*.yml` were one-off diagnostics created while setting up a
specific runner (confirming Docker/toolchain/network availability,
permission issues, etc.) — they are not part of the pipeline itself and
are safe to delete once a runner has been verified.
