# codiqo-showcase

[![Guava](https://github.com/codiqo/codiqo-showcase/actions/workflows/project-guava.yml/badge.svg)](https://github.com/codiqo/codiqo-showcase/actions/workflows/project-guava.yml)

Codiqo analysing open-source projects in public, on a schedule, with the results published as
read-only dashboards anyone can open.

Nothing here is a fork or a patch of the projects being analysed. Each workflow checks a project
out, builds its recent commits, resolves the call graph of every change, and submits the result to
a Codiqo organization dedicated to that project. The build logs are public, so any number on a page
can be traced back to the run that produced it.

| Project | Workflow | Showcase page |
|---|---|---|
| [google/guava](https://github.com/google/guava) | [`project-guava.yml`](.github/workflows/project-guava.yml) | [codiqo.io/showcase/guava](https://codiqo.io/showcase/guava) |
| [ebean-orm/ebean](https://github.com/ebean-orm/ebean) | [`project-ebean.yml`](.github/workflows/project-ebean.yml) | [codiqo.io/showcase/ebean](https://codiqo.io/showcase/ebean) |
| [jetty/jetty.project](https://github.com/jetty/jetty.project) | [`project-jetty.yml`](.github/workflows/project-jetty.yml) | [codiqo.io/showcase/jetty](https://codiqo.io/showcase/jetty) |
| [EsotericSoftware/kryo](https://github.com/EsotericSoftware/kryo) | [`project-kryo.yml`](.github/workflows/project-kryo.yml) | [codiqo.io/showcase/kryo](https://codiqo.io/showcase/kryo) |

**Contents** — [How a run works](#how-a-run-works) · [Run Codiqo on your own repository](#run-codiqo-on-your-own-repository) · [Add a project to this showcase](#add-a-project-to-this-showcase) · [Project notes: guava](#project-notes-guava) · [Project notes: ebean](#project-notes-ebean) · [Project notes: jetty](#project-notes-jetty) · [Project notes: kryo](#project-notes-kryo) · [Reusable workflow reference](#reusable-workflow-reference) · [What bounds a run](#what-bounds-a-run) · [Troubleshooting](#troubleshooting) · [Repository layout](#repository-layout) · [Contributor privacy](#contributor-privacy)

---

## How a run works

Two files, one job. `project-guava.yml` holds everything specific to one project;
`analyze-project.yml` holds the
policy every project shares and is the only place that talks to the action.

```mermaid
sequenceDiagram
    autonumber
    participant C as Daily cron · 03:17 UTC
    participant P as project-guava.yml
    participant S as analyze-project.yml
    participant R as ubuntu-latest runner
    participant A as api.codiqo.io

    C->>P: schedule, or a manual dispatch
    P->>S: workflow_call: repository, slug, JDKs, engine options
    S->>R: check out google/guava with full history
    S->>R: install JDK 26 — Maven JVM and compile toolchain
    S->>R: run codiqo/codiqo-action
    R->>A: index the commit window
    A-->>R: commits still unanalysed, oldest first
    loop per commit, until the 340 minute budget runs out
        R->>R: build, test, resolve the call graph
        R->>A: submit the analysis
    end
    Note over A: codiqo.io/showcase/guava now shows the result
```

Each commit is a complete build and is submitted on its own, so a run that stops early loses
nothing. Where a commit can leave that loop matters more than the loop itself:

```mermaid
flowchart TD
    idx["Index the commit window<br/>P3M of first-parent history"] --> ask{"Commits still<br/>unanalysed?"}
    ask -->|"no"| green["Run ends green,<br/>nothing to do"]
    ask -->|"yes, oldest first"| build["Build the commit<br/>compile, tests, coverage"]
    build --> analyse["Import with the language server<br/>call graph, copy-paste, diagnostics"]
    analyse --> submit["Submit the analysis"]
    submit --> ask
    build -->|"build fails or times out"| excl["Excluded as BUILD_FAILURE,<br/>never retried"]
    excl --> ask
```

Only recent commits are analysed. Older commits usually cannot be built at all: their plugin
versions and JDK expectations have long since expired, and each one that cannot resolve is
excluded after a single attempt.

## Run Codiqo on your own repository

This is the common case, and it does not need anything from this repository. Your own repository is
the one being measured, so there is no second checkout and no reusable workflow.

1. Get an API key at [codiqo.io](https://codiqo.io) and add it as the repository secret
   `CODIQO_API_KEY`.
2. Commit the workflow below as `.github/workflows/codiqo.yml`.
3. Run it once from the Actions tab and confirm the summary reports a non-zero caller count.

```yaml
name: Codiqo

on:
  schedule:
    - cron: '0 3 * * *'
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: codiqo
  cancel-in-progress: false

jobs:
  analyze:
    runs-on: ubuntu-latest
    timeout-minutes: 340
    steps:
      - uses: actions/checkout@v7
        with:
          # Required: Codiqo skips any commit whose parent is missing locally.
          fetch-depth: 0

      - uses: codiqo/codiqo-action@main
        with:
          api-key: ${{ secrets.CODIQO_API_KEY }}
          # 25 or newer: the plugin declares requiredJavaVersion 25, and Maven refuses to run a
          # plugin whose prerequisite it cannot meet. It must also be able to build your project.
          java-version: '26'
```

Every other option is documented in [`codiqo/codiqo-action`](https://github.com/codiqo/codiqo-action).

## Add a project to this showcase

Use this path only for measuring a repository you do not own, publicly. Copy
[`project-guava.yml`](.github/workflows/project-guava.yml) to `project-<slug>.yml` — one file per
project, `analyze-project.yml` holding everything they share — and change four things:

```yaml
      target-repo: apache/commons-lang     # the repository to analyse
      showcase-slug: commons-lang          # the public page under codiqo.io/showcase/
      jdks: |                              # every toolchain version the project asks for;
        26                                 # the one running Maven goes last, and must be 25+
```

```yaml
    secrets:
      CODIQO_API_KEY: ${{ secrets.CODIQO_API_KEY_COMMONS_LANG }}
```

Then:

1. Create a Codiqo organization for the project, and add its key as a repository secret. One key per
   project keeps a mistake in one workflow away from every other page.
2. Dispatch the workflow from the Actions tab with `max-commits-per-run: 1` and
   `ignore-coverage: true`. That proves the wiring in one build rather than a hundred.
3. Confirm the analysis reports a **non-zero caller count**. Zero callers means the language server
   never built a model, not that the code has no callers.
4. Drop both overrides and let the schedule take over.

Two rules keep the key safe, and both are structural rather than a matter of care: every workflow
here triggers only on `schedule` and `workflow_dispatch`, and `analyze-project.yml` is `workflow_call`
only. No fork and no pull request can reach a secret.

## Project notes: guava

A worked example of what per-project tuning looks like, and why each line exists.

- **One JDK, and it has to be 26.** Three constraints meet at that number: `codiqo-maven-plugin`
  declares `requiredJavaVersion` 25, guava's modules bind `maven-toolchains-plugin` at JDK 26, and
  surefire selects `${surefire.toolchain.version}`, which defaults to the Maven JVM's own release.
  Installing `26` alone satisfies all three, because `setup-java` both sets `JAVA_HOME` and registers
  the toolchain entry. Upstream CI runs Maven on 26 for the same reason.
- **The `android/` mirror is excluded.** It is a file-for-file copy of guava's sources, 1,572 of
  3,227 tracked `.java` files. Copy-paste detection reads the indexed file set, so without the
  exclusion it matched the tree against its own copy and exhausted the heap twice, at 9 GB and again
  at 24 GB. Excluding it took the peak to 4 GB in 2.7 seconds and changed nothing analytically: the
  symbol count was identical, because those files carry no symbols of their own.
- **`guava-gwt` is excluded as a module.** It recompiles guava's and guava-testlib's sources under
  a second artifact, so a fully-qualified name reaches the analysis twice and code-unit attribution
  goes ambiguous. Before this exclusion, four commits with real code changes reported zero code
  units and zero files changed — one of them added a public `MediaType` constant and scored 0.
  Paths cannot express this: a module is excluded by coordinates, not by directory.
- **Heap is set explicitly.** A hosted runner has 16 GB and the JVM claims 25 % of it by default,
  which is 4 GB, while the diagnostics stage alone peaked at 5 GB here. The whole run reached 9.0 GiB
  of 15 on the runner, counting the forked build and the language server.
- **The schedule keeps up.** Guava landed 34 commits in the last 30 days, so once the initial `P3M`
  backlog drains, a daily run has roughly one commit to do.

## Project notes: ebean

The contrasting case: a project that needs almost nothing.

- **JDK 25, the floor rather than a preference.** Ebean compiles to release 11 and upstream CI tests
  on 21 alone, but the plugin's `requiredJavaVersion` is 25. Both 25 and 26 still accept
  `--release 11`, so 25 is the closest supported JDK to what upstream exercises. No toolchain is
  pinned, so one entry covers everything.
- **Docker is part of the build.** Five modules start containers through `ebean-test` — postgis,
  pgvector and a shared redis. A hosted runner provides it, and the image pulls are paid once per
  run rather than once per commit.
- **Nothing to exclude and nothing to tune.** No jacoco, PMD, CPD or SpotBugs of its own, no
  failsafe phase, no mirrored source tree. Every default applies unchanged.
- **Cheap by comparison.** The whole reactor builds and runs 2,669 tests in about three minutes
  locally, against roughly twenty-four minutes per commit for guava.
- **One trap.** Ebean's `default` profile is `activeByDefault` and is what contributes the `tests`
  module. Passing any other profile with `-P` deactivates it, the reactor loses every test, and the
  build still goes green with coverage at zero. Which is the argument for not adding `maven-args`
  here.

## Project notes: jetty

The large case, and the one that needs both the clock and the heap.

- **Size sets every other choice.** Roughly 450 reactor modules and 37,000 tests on `jetty-12.1.x`.
  A full build with `-T 1C` took about half an hour on a 14-core workstation, so a four-core runner
  cannot fit a commit inside the shared hour. `per-commit-timeout` is `2h` and
  `build-timeout-minutes` is `90`, leaving half an hour for the analysis. Without the second, the
  forked build keeps the action's 45 minutes whatever the per-commit deadline says. Both are
  estimates to replace with the figures the first runs log.
- **Every six hours, not daily.** Jetty lands about 49 first-parent commits a month. At two hours
  a commit, one 340-minute run a day falls behind for good, so the cron fires four times a day and
  the concurrency group chains the runs instead of overlapping them.
- **Jetty's own build cache is switched off.** `.mvn/` enables `maven-build-cache-extension`, which
  would restore unchanged modules without running their tests: no coverage, no failure, and a green
  build. The fork does not inherit `-D` user properties, so `maven.build.cache.enabled=false`
  travels in `maven-opts` as a system property, which the extension reads as a fallback.
- **Two modules at a time, not `1C`.** Jetty's surefire `argLine` asks for `-Xms4g -Xmx6g` per
  test fork, and at `1C` up to four modules test at once on a hosted runner. The first run exhausted
  the runner's memory during the build of its first commit and the runner shut down with exit 143,
  so `maven-parallelism` is `'2'`. If memory still runs out, lower it to `1` before touching the heap.
- **JDK 25.** Jetty 12.1 compiles for 17 and pins no toolchain; a full local build on Temurin 25
  preceded this wiring.
- **Duplication is real, not mirrored.** `jetty-ee10` and `jetty-ee11` are maintained side by side:
  931 of the 1,081 `ee10` sources with an `ee11` counterpart are identical once the environment name
  is swapped. Unlike guava's `android/` tree, both are shipped code, so neither is excluded and the
  page reports the duplication as it stands. `jetty-ee8` is generated from `ee9` at build time and
  holds only 30 tracked `.java` files.
- **Jetty's duplication is not comparable with the other pages.** Two settings differ, both forced by
  this codebase. `cpd-ignore-identifiers` is `false`, so only clones that keep their names count:
  PMD's Java tokenizer crashes on this tree when it replaces identifiers
  ([pmd/pmd#7133](https://github.com/pmd/pmd/issues/7133)). And `cpd-minimum-tile-size` is `125`,
  not `100`: complete copy-paste detection ran out of the analysis heap with the index already
  holding 6 GB, and fewer, longer clones take less to hold.
- **10 GB of heap, not 8.** `maven-opts` is `-Xmx10g`, for the analysis, the language server and,
  since `fork-maven-opts` is left empty, the forked build too.
- **Environment-sensitive tests do not wedge a commit.** Some tests need Docker images, a remote
  snapshot repository or native QUIC. The plugin runs the fork with `maven.test.failure.ignore`, so a
  failing test costs its own coverage rather than the commit.

## Project notes: kryo

The small case, and the one whose only trap is a module that builds the same sources twice.

- **JDK 25, the floor rather than a preference.** Kryo compiles its main sources for Java 8, builds
  upstream on 11 and tests on every LTS through 25 and on 27. The plugin's `requiredJavaVersion` is
  25, which upstream already tests on. A full local build on Temurin 25 passed all 337 tests in about
  twenty seconds. No toolchain is pinned, so one entry covers everything.
- **`kryo5` is excluded as a module.** `main-versioned/` declares the same `src/` and `test/`
  directories as `main/` and shades the result into `com.esotericsoftware.kryo.kryo5`. The local
  build compiled the same 77 sources and ran the same 337 tests in both modules, so every
  fully-qualified name would reach the analysis twice — the guava-gwt case again, and excluded the
  same way, by coordinates. The fork still builds and tests `kryo5`; the duplicated work costs about
  ten seconds a commit.
- **`benchmarks` stays in.** It is a real reactor module with its own JMH sources, not a copy.
- **Nothing else to tune.** Every shared default applies: the hour per commit, `1C` parallelism and
  the `-Xmx8g` heap all have ample room. Dependabot accounts for 9 of the 43 first-parent commits in
  the last three months, and the bot filter keeps them off the page.

## Reusable workflow reference

`analyze-project.yml` exposes only what a project genuinely varies. Everything else is either showcase
policy, fixed on the action step, or an action default left alone.

| Input | Default | Purpose |
|---|---|---|
| `target-repo` | *required* | Repository to analyse, as `owner/name`. |
| `target-ref` | default branch | Ref to analyse. Must name a branch. |
| `showcase-slug` | *required* | Public page slug, also the concurrency key. |
| `jdks` | `26` | JDKs to install, one per line. All become toolchain entries; the **last** runs Maven, the tests and the language server, so it must be **25 or newer** — `codiqo-maven-plugin` declares `requiredJavaVersion` 25. |
| `commit-window` | `P3M` | How far back to index. See [what bounds a run](#what-bounds-a-run). |
| `max-commits-per-run` | *(empty: unbounded)* | Cap on commits per run. Empty takes every pending commit, oldest first. |
| `per-commit-timeout` | `1h` | Deadline for one commit, build and analysis together. |
| `build-timeout-minutes` | `45` | Deadline for the forked build of one commit, tests included. It has to fire before `per-commit-timeout`, so the action clamps it to three quarters of that. |
| `maven-opts` | `-Xmx8g` | Heap for the analysis. The default 25 % of runner RAM — 4 GB — is not enough: guava's diagnostics stage alone peaked at 5 GB. The language server copies its `-Xmx`, and the forked build inherits it unless `fork-maven-opts` is set, so it is not the only claim on the runner. |
| `fork-maven-opts` | *(empty: inherits `maven-opts`)* | `MAVEN_OPTS` for the forked build alone, for a project whose analysis needs more heap than its build. It replaces `maven-opts` for the fork whole, so repeat any `-D` the project's build relies on. |
| `maven-user-properties` | *(none)* | `key=value` lines passed as `-Dkey=value`, for project-specific engine options. They do not reach the forked build; use `maven-opts` for a property the project's own build must see. |
| `maven-parallelism` | `1C` | Maven `-T` for the per-commit build, one thread per runner core. The plugin hands it to the fork. Each concurrently built module may start its own test JVM, so lower it if a commit dies with exit 137. |
| `ignore-coverage` | `false` | Skip tests. Faster, and produces no coverage data. |
| `cpd-ignore-identifiers` | `true` | Match clones whose identifiers were renamed. `false` counts only clones that keep their names. |
| `cpd-minimum-tile-size` | `100` | Shortest reported clone, in tokens. PMD-CPD's own default, and SonarQube's Java sensitivity; the engine's 64 reports shorter clones than either tool. A project may raise it when copy-paste detection cannot otherwise fit in memory, and its notes must say so. |

Fixed as policy, and deliberately not configurable per project:

| Setting | Value | Why |
|---|---|---|
| `fail-on-jdtls-error` | `true` | A failed import leaves every caller count at zero, which is indistinguishable from "no callers" once stored. A red run is recoverable; a page asserting a blast radius of zero is not. |
| `stop-on-first-failure` | `false` | The pending list is walked oldest-first, so one wedged commit must not block every newer one. |
| `agent-instructions` | `false` | The analysed project never opted in. Its `AGENTS.md` must not steer what we publish about it. |
| `log-commit-authors` | `false` | Logs on a public repository are public. |
| `exclude-author-emails` | `*[bot]@*` | A showcase measures human work. Narrower than the engine's `*bot*` on purpose: it catches GitHub app identities without catching a human whose address merely contains the word — guava's Google-internal changes are exported under such an address. |
| `persist-credentials` | `false` | The analysed project's own build runs over that checkout. |

## What bounds a run

What bounds a run is a clock, not a commit count.

| Bound | Value | Effect |
|---|---|---|
| Job timeout | 340 min | The real limit. Six hours is the hosted-runner maximum, and stopping short of it means Actions cancels the job, which still uploads the logs and writes the summary. |
| Per-commit deadline | 60 min | The build and test budgets are fitted under it. A commit that exceeds them is excluded as a build failure and not retried. Jetty raises it to 2 h. |
| Commit window | `P3M` | Bounds what enters the index. It does **not** filter the pending list afterwards. |
| Commits per run | unbounded | Every pending commit, oldest first, until the job timeout. |

Two consequences worth internalising before the first run:

- **Widening the window queues a backlog.** `P3M` over guava enqueues on the order of 100 commits,
  and one run clears whatever fits in 340 minutes. The rest stay pending, so the schedule works
  through them over the following days. Nothing is lost, and nothing is analysed twice.
- **A cancelled run is not a failed analysis.** Every commit is submitted independently, so the
  pages fill in as the backlog drains, even while the catch-up runs are ending as cancelled.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Summary reports zero commits pending | Shallow clone: Codiqo skips any commit whose parent is missing locally | `fetch-depth: 0` on the checkout |
| Analysis completes, every caller count is zero | The language server never imported the project | Read the import log in the run artifact. `fail-on-jdtls-error: true` turns this into a red run rather than a quiet zero |
| `The plugin ... has unmet prerequisites: Required Java version 25` | The JDK running Maven is older than the plugin allows | Make the **last** entry in `jdks` 25 or newer |
| Build fails with no toolchain found for a JDK | The project pins a toolchain the runner does not have | List every version the project asks for in `jdks`, Maven's own last |
| Killed during copy-paste detection at very high heap | The repository mirrors its own sources, so the tree matches against its copy | `maven-user-properties: codiqo.excludePaths=<tree>/**` |
| Commit killed with exit 137, or the job ends with exit 143 and "The runner has received a shutdown signal" | The kernel ran out of memory: the forked build, its concurrent test JVMs and the language server together exceeded the runner | Lower `maven-parallelism`, for example to `2` |
| Job cancelled after 340 minutes | Backlog larger than one run's budget | Expected during a catch-up. It resumes on the next run |
| Everything appears pending again | The commit index is keyed by branch name | `analyze-project.yml` reads the branch from the checkout, so the analysed repository's name is used rather than this repository's |

Logs for every run are uploaded as an artifact named `codiqo-logs-<slug>`, kept for seven days, with
secrets redacted before upload.

## Repository layout

```
.github/workflows/
├── analyze-project.yml    shared policy, workflow_call only, never triggered directly
├── project-guava.yml      one project: schedule, repository, slug, JDKs, engine options
├── project-ebean.yml      another, needing nothing but a different JDK
├── project-jetty.yml      a large reactor: longer deadline, six-hourly schedule, build cache off
├── project-kryo.yml       a small reactor with one module that recompiles another's sources
└── project-<slug>.yml     ... one file per project, added the same way
```

GitHub reads workflows only from the top level of `.github/workflows`, so the convention carries the
grouping that folders would: `analyze-` is the shared machinery, `project-` is one project each.
Adding a project adds one `project-` file, and `analyze-project.yml` changes only when the policy
does.

## Contributor privacy

The projects analysed here did not ask to be measured, and their authors never opted in. So the
public pages publish the work, not the person:

- Contributor names are masked to initials — `Jane Doe` is served as `J*** D***` — and git
  addresses never leave the server.
- Commit messages are scrubbed of attribution trailers (`Co-authored-by`, `Signed-off-by`,
  `Reviewed-by`, `Reported-by`, `Acked-by`, `Tested-by` and the rest) and of any inline address.
- Seniority judgements, account data and internal HR structure are withheld entirely in public
  mode. The server is the enforcement boundary, not the client.
- CI logs are public, so author names and emails are kept out of them too.

This is not anonymity, and it is not presented as such. Commit SHAs remain on the page and resolve
to their real author upstream. What masking removes is the name from our pages and from search
indexing.

If you maintain a project analysed here and would rather it were not, open an issue and it will be
removed.
