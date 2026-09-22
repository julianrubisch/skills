# PR Tour Guide

Phases 0 to 5. Phases 0 to 3 are mechanical and should be run without
commentary. Phase 4 is where judgement goes. Phase 5 publishes.

Working files go in the scratchpad directory, never in the repository.

## Phase 0: Resolve and fetch

1. Parse the argument. URL or number → `HOST`, `PR` (see `hosts.md`). No
   argument → git-only mode: `BASE=main` (or the upstream default),
   `HEAD=HEAD`, no host links, no comment.
2. Fetch metadata: title, body, base/head refs and SHAs, URL, commit list.
3. Fetch the file list with additions/deletions and status
   (added/modified/deleted/renamed). In git-only mode:
   `git diff --numstat --diff-filter=ADMR -M $BASE...$HEAD` plus
   `git diff --name-status -M $BASE...$HEAD`.
4. `git fetch origin $BASE $HEAD` so `git show $HEAD:path` works for every
   file.

Stop and tell the user if the diff has fewer than 6 files. A tour of 5 files
is alphabetical order with extra steps; suggest reviewing it directly.

## Phase 1: Noise filter

Move these to the skip list, with the reason, before doing anything else:

| Pattern | Reason |
|---|---|
| `Gemfile.lock`, `yarn.lock`, `package-lock.json`, `pnpm-lock.yaml`, `bun.lockb` | lockfile |
| `db/schema.rb`, `db/structure.sql` | generated from migrations; read the migration instead |
| `app/assets/builds/**`, `public/assets/**`, `public/packs/**`, `*.min.js`, `*.min.css`, `*.map` | build output |
| `test/fixtures/**`, `spec/fixtures/**`, `spec/cassettes/**`, `test/vcr_cassettes/**` | fixture data |
| `**/*.svg`, `**/*.png`, `**/*.jpg`, `**/*.woff2`, other binaries | binary |
| `config/credentials/*.enc`, `*.key` | never open in a review tool |
| `vendor/**`, `node_modules/**` | vendored |
| `**/*.po`, `config/locales/*.yml` with only key additions | translations (keep if a locale file is *the* change) |
| `.rubocop_todo.yml`, `coverage/**` | tooling residue |

Do not skip `db/migrate/**`, `config/routes.rb`, `config/initializers/**` or
`.github/workflows/**`; those are the files a reviewer must see first, not
last. The skip list is still rendered at the bottom of the tour so nothing
disappears.

### Phase 1b: Triage (ask for large PRs)

A 60-file PR is not 60 files of the same weight. When more than 30 files
survive the noise filter, or `--triage` was passed, ask once with
`AskUserQuestion`:

- **Triage** (recommended above 30 files): split the tour into chapters
  that need eyes and one "Safe to skim" chapter at the end for mechanical
  changes. Every file stays in the tour, keeps its link and its checkbox,
  and the skim chapter has a "mark all as read" button.
- **Full tour**: every file gets a chapter and a "why".

What qualifies as safe to skim, each with its test:

| Kind | Test |
|---|---|
| Rename without content change | `git diff -M90% --name-status` shows `R100`, or `R9x` with only the path in the diff |
| Formatting only | `git diff -w --ignore-blank-lines $BASE $HEAD -- path` is empty, or the change is a formatter run (`.rubocop.yml` touched in the same commit, every hunk is whitespace or quotes) |
| Mechanical mass edit | the same one-line change in 5 or more files (a constant or method rename, a namespace move); keep **one** representative in the real chapters with `why` naming the others, put the rest in skim |
| Generated or vendored code that passed the noise filter | `bin/rails g` output, `schema.rb` variants, minified assets |
| Additions of translation keys, fixtures, seeds | no logic; the file is pure data |
| Deleted dead code | a deleted file with no remaining references at `$HEAD` (`git grep -l <constant>` empty) |

Never skim: anything with `risk.level` 2, any migration, any file in
`config/`, any deleted file that still has references, and any test whose
subject is in a real chapter. When in doubt, it is not safe to skim.

The skim chapter: `title` "Safe to skim", `skim: true`, `lede` says what
kinds are in it and how many of each, and every file's `why` is its
reason in three to six words ("rename only", "formatter run", "same
rename as user.rb"). Order inside by kind, then path.

Say in the summary how the split came out ("41 files: 17 to read, 24
safe to skim").

## Phase 2: Signals

### 2a. Rails layer

Assign a layer per remaining file by path. The rank is the default reading
order between chapters; lower reads first.

| Rank | Layer | Paths |
|---|---|---|
| 0 | `infra` | `.github/**`, `.gitlab-ci.yml`, `Dockerfile*`, `.devcontainer/**`, `bin/**`, `config/environments/**`, `config/application.rb`, `config/*.yml` |
| 1 | `deps` | `Gemfile`, `package.json` |
| 2 | `migration` | `db/migrate/**`, `db/seeds*` |
| 3 | `config` | `config/initializers/**`, `config/routes.rb`, `config/locales/**` (if not skipped) |
| 4 | `model` | `app/models/**`, `lib/**` (domain code) |
| 5 | `job` | `app/jobs/**`, `app/mailers/**`, `app/channels/**` |
| 6 | `controller` | `app/controllers/**`, `app/policies/**`, `app/forms/**`, `app/queries/**` |
| 7 | `view` | `app/views/**`, `app/components/**`, `app/helpers/**`, `app/javascript/**`, `app/assets/stylesheets/**` |
| 8 | `test` | `test/**`, `spec/**` |
| 9 | `docs` | `*.md`, `docs/**` |

Anything unmatched: `other`, rank 5, and mention it in the file's "why".

### 2b. Symbol references between changed files (Ruby)

Build a directed edge list `A -> B` meaning "A references something B
defines". This is what the ICSE survey respondents asked for and what the
within-chapter order comes from.

For each changed `.rb` file at `$HEAD`, collect what it **defines**:

```bash
git show "$HEAD:$path" | grep -oE '^\s*(class|module)\s+[A-Z][A-Za-z0-9_:]*' | awk '{print $2}'
```

The same grep works for other typed languages with their keywords: Swift
and Kotlin `(class|struct|enum|protocol|interface|extension|object)`,
JavaScript/TypeScript `(class|interface|type)` plus `export function`.
The Rails conventions below do not apply there; the directory is the
grouping signal instead (Phase 4a absorbs by layer, and for a non-Rails
repository the layer is the top-level directory: `Sources/<Module>`,
`Tests/<Module>`, `app/`, `lib/`).

Then, for each other changed file (any extension), test whether it
**references** any of those constants, as a whole word:

```bash
git show "$HEAD:$other" | grep -qwE "$constant"
```

Add the Rails conventions the grep will not see:

- `has_many :orders`, `belongs_to :user` → references `Order`, `User`
- `app/controllers/orders_controller.rb` → `app/views/orders/*`, and
  `app/policies/order_policy.rb`
- `config/routes.rb` → every changed controller
- a migration → the model whose table it touches (`create_table :orders`,
  `add_column :orders, …` → `Order`)
- `app/views/**/_thing.*` ← any changed view that renders `thing`
- test files → the file they test, by path mirror
  (`test/models/user_test.rb` → `app/models/user.rb`)

The edge direction is "reads after": when A references B, read B first.

If `attractor` reports `symbols` in its JSON (attractor-javascript ≥ 0.4.0
does for JS; Ruby is tracked in julianrubisch/attractor#130), use those
instead of the grep for that language.

### 2c. Commit structure

```bash
git log --reverse --format='%h %s' --stat=200 $BASE..$HEAD
```

Commits are **coherent** when most touch fewer than ~8 files, subjects are
descriptive, and file sets overlap little. Then the commit sequence is the
author's own narrative and becomes the tiebreaker inside chapters. Squashed
or "wip" histories carry no information; ignore them and say so in the
summary ("history is one squashed commit; order is derived").

## Phase 3: Risk

Purpose: one badge per file so the reviewer knows where to slow down. It
never changes the order.

Two inputs, from two sides of the change:

- **What the PR added**: complexity delta per file (attractor, or flog on
  both refs). Complexity the file already had is not the PR's risk; a
  two-line edit to a 300-flog model is not where the bug is.
- **How the PR got there**: churn *inside* the PR. A file touched in 7 of
  9 commits is one the author struggled with (unsettled design, repeated
  fixes). This is the strongest "look here" signal available, it is not
  what attractor's historical `churn` measures, and it needs only git:

```bash
COMMITS=$(git rev-list --count $BASE..$HEAD)
for f in $files; do
  printf '%s\t%s\n' "$(git rev-list --count $BASE..$HEAD -- "$f")" "$f"
done | sort -rn > tour-churn.tsv
```

Skip this when history is squashed (`COMMITS` ≤ 2): every file scores 1
and the signal is gone; say so in the summary.

Historical hotspot status (attractor's `refactor_head`) is a modifier,
not an input: it says the terrain is fragile, not that this change is.

### Preferred: attractor

#### Finding attractor

Version managers select a Ruby per directory. The project's `.ruby-version`
usually names a Ruby that does *not* have attractor installed, while some
other Ruby on the machine does. `command -v attractor` inside the project
then fails, and the tour silently loses its complexity badges. Do not
accept that. attractor does not need the project's Ruby: it reads files and
runs git and flog, so any installed Ruby with the gems will do.

Resolve in this order, and stop at the first hit:

1. **The project's Ruby has it**: `gem list -i attractor` in the project
   directory. `ATTRACTOR="attractor"`.
2. **Another Ruby has it**: check every installed Ruby.

   ```bash
   # rbenv
   for v in $(rbenv versions --bare); do
     RBENV_VERSION=$v gem list -i attractor >/dev/null && echo $v
   done
   # mise
   for v in $(mise ls ruby --json | jq -r '.[].version'); do
     mise x ruby@$v -- gem list -i attractor >/dev/null && echo $v
   done
   # asdf: ASDF_RUBY_VERSION=$v, same loop over `asdf list ruby`
   ```

   Then `ATTRACTOR="env RBENV_VERSION=$v attractor"` (or
   `mise x ruby@$v -- attractor`, or `env ASDF_RUBY_VERSION=$v attractor`).
3. **No Ruby has it**: install into the project's Ruby, from the project
   directory so the manager picks the same version the user gets:

   ```bash
   gem install attractor attractor-ruby attractor-javascript
   # plus the plugin for each other language in the diff: attractor-swift, ...
   ```

   Tell the user in one line that you did this and for which Ruby. It is a
   change to their gem set, not to the repository, so it is allowed; it is
   not silent.

If the Bash tool cannot see `rbenv`/`mise`/`asdf` at all, the shell is
missing the login PATH. Rebuild it the way `jr-rails-second-opinion` does
(probe fish/zsh/bash login shells, merge their PATHs, add `~/.rbenv/shims`,
`~/.local/share/mise/shims`, `~/.asdf/shims`), then retry from step 1.

Whatever `$ATTRACTOR` resolved to, check the plugin gems under the **same**
Ruby before running: `$ATTRACTOR version` proves the core loads, and the
`gem list -i` calls in the chosen environment prove the calculators are
there. Only fall back to flog when steps 1 to 3 all failed, and say which
step failed.

#### Running it

```bash
$ATTRACTOR version
git diff --name-only $BASE...$HEAD \
  | $ATTRACTOR diff --base $BASE --head $HEAD --files - -c 0 --format json > tour-attractor.json
```

`-c 0` is required. The default minimum churn is 3, which drops every file
with fewer than 3 commits in its history: all new files in the PR, and most
of the rest in a young repo. The result is a diff with no rows and no
error.

The language calculators are separate gems and attractor loads only the
ones installed: without `attractor-ruby` there are no rows for Ruby files,
again with no error. Check both before the run, not after.

`attractor diff` checks both refs out into temporary worktrees, so it works
regardless of what the user has checked out, and cleans up after itself.
Rows carry `complexity_base`, `complexity_head`, `delta`, `churn`,
`score_head`, `refactor_base`, `refactor_head`, and `details_head` with
per-method `{score, line, end_line}`. Requires attractor ≥ 2.7 with
attractor-ruby ≥ 0.4 (and attractor-javascript ≥ 0.4 for JS). Other
languages have their own plugin gem on attractor ≥ 2.8, scored with
lizard through `uvx` (attractor-swift; Kotlin, Go and others follow the
same template): check `gem list attractor-` against the extensions in the
diff, and install the missing plugin the same way as the Ruby one. A
language with no plugin gets no `churn` and no complexity, and falls to
the diff-size rule below for that file only.

Badge levels from the rows:

| Level | Label | Rule |
|---|---|---|
| 2 | `reworked` | touched in ≥ 50% of the PR's commits, with `COMMITS` ≥ 4 |
| 2 | `grew` | `delta > 20` |
| 1 | `grew` | `delta > 5`, or `complexity_base` nil and `complexity_head > 30` (large new file) |
| 1 | `reworked` | touched in ≥ 3 commits |
| 0 | none | everything else, including improvements (`delta < 0`; note those in "why", reviewers like to see them) |

Then apply the modifier: `refactor_head` true (top percentile of churn ×
complexity in the repo) raises the level by one, to a maximum of 2, and
the label becomes `hotspot`. When two rules fire, keep the higher level and
the label of the rule that produced it. Put the numbers in the badge text
so the reviewer sees why: `complexity_head` (and `complexity_base` when it
exists) in `risk.complexity_*`, and the commit count in `risk.commits` as
`"7/9"`.

Also carry the churn numbers, for the scatter the page draws (below):
`risk.churn` is attractor's head `churn`, and `risk.churn_base` is
`churn` minus the file's in-PR commit count (the PR's own commits are the
difference between the two refs). Files without a complexity score
(deleted, non-Ruby without attractor-javascript) get no `churn` and stay
off the chart.

### The churn × complexity scatter (ask first)

With `risk.churn`, `risk.churn_base`, `risk.complexity_base` and
`risk.complexity_head` on the files, the page can draw each touched file
as a trace from its base position to its head position on a churn (x)
against complexity (y) plane: the same plot attractor's report shows,
restricted to the PR. A file that moved up added complexity; a file far
right changes often; a hotspot is labelled. Real repositories are
heavy-tailed on both axes, so the page offers linear and log scales
(log uses `log10(v + 1)`, so new files with zero base churn keep a
place), starts in log when the largest value is more than 8 times the
median on either axis, and remembers the reviewer's choice.

It is opt-in. When attractor ran, ask once, with `AskUserQuestion`, before
Phase 4:

- **Traces only** (recommended): the touched files. No extra work.
- **Traces with the repository as context**: adds the other files as
  faint points, so the reviewer sees whether a touched file sits among
  the hotspots or in a quiet corner. Costs one `attractor calc -c 0
  --format json` in a head worktree (`git worktree add`, then remove);
  say how long the diff run took, the calc run takes about as long. Fill
  `landscape` as `[{path, churn, complexity}]` for files **not** in the
  PR, capped at 400 by score.
- **No chart.**

Set `scatter: true` only on the first two answers; the page draws nothing
without it. Do not ask when attractor did not run (flog and diff-size
fallbacks have no churn), and do not ask again on a re-run of the same
PR; keep the previous answer.

When `details_head` names a method with a score above 30, put it in the
file's `look_for` with its line and, if `HOST` is set, pass `line.new` so
the link lands on the method.

### Fallback 1: flog only

If `attractor` is missing but `flog` is (`gem list -i flog`), score the
head version of each Ruby file in a temporary worktree:

```bash
git worktree add -q "$SCRATCH/head" $HEAD
(cd "$SCRATCH/head" && flog -s $files)   # totals per file
git worktree remove -q "$SCRATCH/head"
```

Level 1 for a total above 60, level 2 above 120 (label `complex`), no
delta. In-PR churn applies unchanged. Say in the summary that complexity
is head-only.

### Fallback 2: diff size

Neither tool: level 1 when `additions + deletions > 200` for a single file,
level 2 above 500 (label `large`). In-PR churn applies unchanged. Say so.

The fallbacks exist for machines without Ruby gems at all (a JS-only
checkout, a CI runner). On a developer machine, "Finding attractor" step 3
installs it; do not fall back there.

## Phase 4: Chapters

### 4a. Mechanical pass

1. **Cluster** the non-skipped files by union-find over the Phase 2b edges.
   A test joins its subject's cluster. Files with no edges are singletons.
2. **Order chapters** by the minimum layer rank in each cluster; ties by
   size, larger first (the big cluster is usually the feature).
3. **Order files inside a chapter** topologically over the edges
   (dependencies first). Break ties by layer rank, then commit order when
   coherent, then path.
4. **Absorb singletons**: a single file with no edges joins the chapter
   whose dominant layer matches its own; if none, they form a final "Around
   the edges" chapter, which is the one legitimate catch-all and should be
   short.
5. **Split** any chapter above ~12 files by layer (data, domain, web) with
   the same internal order.

### 4b. Judgement pass

This is the step that makes the tour worth opening. For each chapter:

- **Name it** after the thing it contains, with the code's names: "The
  ImportRun record", "Billing address on Order", "The feature flag
  checks". Not "Models" or "Chapter 3".
- **Write a lede**: one or two sentences saying what the reviewer will
  understand after reading it and how it connects to the previous chapter.
- **Reshape** when the mechanics are wrong: merge two chapters that are one
  concern, move a file the grep mis-attributed, pull a file forward when it
  is the entry point even though nothing references it (a new route, a job
  `perform`).
- Order chapters from data to behaviour to web. In a Rails PR this is
  usually migration → model → job/controller → view → tests. A PR that
  renames one concept everywhere reads better grouped by concept. A PR
  that is a refactor reads better old → new.

For each file:

- **`why`**: one line on why it sits here. Refer to neighbours by name
  ("defines `ImportRun`, used by the next two files"). Trace one path end
  to end where you can.
- **`look_for`**: one line, neutral, on what deserves attention: a method
  the risk pass flagged, a public interface change, a callback added, a
  migration that is not reversible, a test that was deleted. This is not
  a finding. If nothing stands out, omit it.

Deleted files go at the end of their chapter with `why` explaining what
replaced them. Renames appear once, under the new path, with the old path
in `why`.

### Writing rules

All prose in the tour (summary, chapter titles, ledes, `why`, `look_for`)
follows ASD-STE100 Simplified Technical English, adapted for code. The
reader is scanning between two windows; write for that.

- One idea per sentence. At most 20 words. Active voice, present tense.
- Instructions start with the verb: "Read the migration first."
- Use the code's own names as the vocabulary: `ImportRun`, `advance!`,
  `import_runs`. Use the same word for the same thing every time. No
  synonyms for variety.
- No metaphors, idioms, or colour: not "hangs off", "walks the file",
  "plumbing", "surface", "story". Say "uses", "reads each row", "the web
  part".
- Noun strings of at most three words. Keep the articles.
- Numbers and names instead of adjectives: not "a large change", but "14
  files"; not "complex", but "flog score 41".
- State facts. No hedging ("seems", "probably"), no praise ("nice",
  "clean"), no drama ("dangerous", "scary").
- Chapter titles are noun phrases that name the thing: "The ImportRun
  record", "The background job". Ledes have at most three sentences.
  Summaries have at most six.

Before publishing, read every `why` once. If a sentence needs the reader
to know a figure of speech, rewrite it.

### 4c. Summary

Two to four sentences at the top: what the PR does in the reviewer's terms,
the shape of the change (n files, which layers, new vs modified), whether
history was usable, and which chapter carries the risk. Then a suggested
split of time if the tour is long ("chapters 1 and 2 are 70% of the
reading").

### 4d. Routes to QA on the dev server

Reading the diff shows what changed; opening the pages shows whether it
works. For each chapter, list the URL paths a reviewer can open on a
running dev server to see the change. Order them like the chapters.

**Where the paths come from**, in order of reliability:

1. **`bin/rails routes`**, when the app can boot with the PR's code. Check
   `git rev-parse HEAD` against `$HEAD`: if the working tree is on the PR,
   run in place; otherwise run in a temporary worktree only if
   `bundle check` passes there (no installs). Filter per changed
   controller:

   ```bash
   bin/rails routes -c ImportRunsController --expanded
   ```

   Take `Verb`, `URI`, and `Controller#Action`.
2. **Static conventions**, when it cannot boot. From `resources`,
   `resource`, `namespace`, `scope`, `get/post/…` lines in `config/routes.rb`
   at `$HEAD`, plus the standard seven actions for `resources`. Mark these
   `"source": "routes.rb (static)"` so the reviewer knows they are derived.
3. **Changed views without a changed controller**: map the view path to
   its action (`app/views/import_runs/show.html.erb` → `ImportRuns#show`)
   and resolve the path with 1 or 2. Partials map to every view that
   renders them; list the first one only.
4. **Rails previews**: a changed mailer with a preview →
   `/rails/mailers/<mailer>/<method>`; a changed component with a preview
   (ViewComponent, Lookbook) → `/rails/view_components/<component>/<preview>`
   or `/lookbook`. Skip when no preview exists; say so in the note.

**Rules:**

- Only `GET` paths are links. For `POST/PATCH/DELETE`, list the `GET`
  page that holds the form ("Submit the form on /import_runs/new") and the
  action it hits.
- Paths with parameters should open. A placeholder (`/import_runs/:id`)
  is a dead link; find a real value and put the concrete path in
  `example` (the page links the example and shows the pattern beside it).
  In order: a fixture or seed that names one (`test/fixtures/*.yml`,
  `db/seeds.rb`); a read-only query against the development database
  when the app boots, `bin/rails runner 'puts ImportRun.order(:id).last&.id'`
  (inside the devcontainer when there is one; on the host this reads the
  reviewer's own database, which is fine, it writes nothing); else keep
  the placeholder and say in `note` what to substitute. Never invent an
  ID; a wrong one gives a 404 that looks like a bug in the PR.
- Jobs, models, and concerns have no path. Put the trigger in `note`
  instead: "Runs after a POST to /import_runs; watch the log or
  /jobs (Mission Control) if mounted."
- At most 8 paths for the whole tour. Prefer the entry page of each
  chapter over every sub-page.
- Include the `note` in Simplified Technical English: what the reviewer
  should see, in one sentence.

**Where the server runs.** Look for `.devcontainer/devcontainer.json`.
If the project has one, that is how the project runs, and the tour should
use it. If not, the project runs on the host; do not suggest adding a
devcontainer to the repository, a review is not the place to change how a
project is run. A throwaway one outside the repository is fine (below).

*With a devcontainer:* start it.

```bash
docker info >/dev/null 2>&1 || echo "Docker is not running"
command -v devcontainer || echo "npm i -g @devcontainers/cli"
devcontainer up --workspace-folder .
devcontainer exec --workspace-folder . bin/setup --skip-server   # or bin/rails db:prepare
devcontainer exec --workspace-folder . bin/dev                    # run in the background
```

Then check how port 3000 reaches the host. Read `devcontainer.json` (and
the compose file it names in `dockerComposeFile`):

- `appPort`, `runArgs` with `-p`, or a compose `ports:` entry publish the
  port. `base_url` is `http://localhost:<port>`.
- Only `forwardPorts`: the devcontainer CLI ignores it; editors honour
  it. Tell the reviewer to open the folder in their editor and choose
  "Reopen in Container", which forwards the port, and keep
  `http://localhost:3000` as `base_url`. Say this in `qa.start`.

The web command comes from `Procfile.dev` or `bin/dev` (`web:` line); the
port from `config/puma.rb` or `.env*`; allowed hosts from
`config/environments/development.rb` (`config.hosts`). Set
`qa.environment` to `devcontainer` and `qa.start` to the exact commands
you ran or the reviewer must run.

*Without a devcontainer:* do not start anything on the host. A host
server runs the PR's migrations against the reviewer's development
database, which is their call to make, not the tour's. Put the host
command in `qa.start` (`bin/dev`, default `http://localhost:3000`) and set
`qa.environment` to `host`. Mention the migrations in the lede only when
the PR contains any: "The PR has 2 migrations. `bin/dev` runs them on your
development database."

Then, after publishing, offer a throwaway container once: the skill can
generate a devcontainer in the scratchpad and run the checkout with
`devcontainer up --config`, so the repository stays untouched and nothing
is proposed to the project. Recipe in `qa-devcontainer.md`. On a yes,
update `qa.*` and republish.

Do not open the paths yourself. That is the reviewer's step; the tour
hands them the list. The page lets the reviewer change the base URL and
remembers it.

### Tour JSON

The template renders this object from a `<script id="tour-data"
type="application/json">` block. Fill every field; the page does no
computation beyond anchors and progress.

```json
{
  "host": "github",
  "pr": {
    "number": 128, "url": "https://github.com/o/r/pull/128", "repo": "o/r",
    "title": "Import runs: background CSV import with progress",
    "base": "main", "head": "feature/import-runs",
    "additions": 812, "deletions": 143, "files": 31, "commits": 9
  },
  "summary": "…",
  "risk_source": "attractor diff 2.7.0",
  "chapters": [
    {
      "title": "The ImportRun record",
      "lede": "All other files use this table and this model. Read the migration first to see the columns.",
      "files": [
        {
          "path": "db/migrate/20260901120000_create_import_runs.rb",
          "layer": "migration", "status": "added",
          "additions": 18, "deletions": 0,
          "why": "Creates the import_runs table with a status column and counters. All files that follow use these columns.",
          "look_for": "The migration has no down method. Make sure that it is reversible.",
          "risk": {"level": 0}
        },
        {
          "path": "app/models/import_run.rb",
          "layer": "model", "status": "added",
          "additions": 96, "deletions": 0,
          "why": "Defines ImportRun and its status changes. The job and the controller use it.",
          "look_for": "The method advance! has a flog score of 41 (line 58).",
          "risk": {"level": 2, "label": "reworked", "complexity_head": 71.2, "commits": "7/9"},
          "line": {"new": 58}
        }
      ]
    }
  ],
  "skipped": [
    {"path": "db/schema.rb", "reason": "generated from migrations"},
    {"path": "Gemfile.lock", "reason": "lockfile"}
  ],
  "qa": {
    "base_url": "http://localhost:3000",
    "environment": "devcontainer",
    "start": "devcontainer up --workspace-folder . && devcontainer exec --workspace-folder . bin/dev",
    "migrations": 1,
    "source": "bin/rails routes",
    "routes": [
      {"verb": "GET", "path": "/import_runs/new", "action": "ImportRuns#new", "chapter": 3, "note": "The upload form. Choose a CSV file and submit it."},
      {"verb": "POST", "path": "/import_runs", "action": "ImportRuns#create", "chapter": 3, "via": "/import_runs/new", "note": "Submit the form. The response redirects to the progress page."},
      {"verb": "GET", "path": "/import_runs/:id", "example": "/import_runs/1", "action": "ImportRuns#show", "chapter": 3, "note": "The counters increase while the job runs."}
    ]
  },
  "generated_at": "2026-09-18T15:04:00+02:00"
}
```

A chapter with `"skim": true` renders collapsed at the end with a "mark
all as read" button (Phase 1b); it is always the last chapter.
`risk.churn` and `risk.churn_base` are numbers (see Phase 3); with
`complexity_*` they feed the scatter, which the page draws only when
`scatter` is `true` (the user's answer). `landscape` is optional:
`[{path, churn, complexity}]` for files not in the PR. `qa` is optional;
omit it when the PR has no web surface. `qa.source` is
`bin/rails routes` or `routes.rb (static)`. `qa.environment` is
`devcontainer` or `host`; `qa.start` is either a bare command (`bin/dev`,
rendered as "Start it with `bin/dev` on the PR branch") or, when more
needs saying, one or two full sentences ending in a period, rendered
verbatim ("A dev server for this branch already runs on port 3100 from
the wa-jquery-teardown worktree. Otherwise run bin/dev there.");
`qa.migrations` is the number of migrations in the PR (omit when 0). Each
route has its own checkbox; ticks are stored and synced like the file
ticks but do not count toward reading progress. `chapter` is the 1-based
chapter the route belongs to. `via` on a non-GET route names the GET page
that holds the form. `host` is `null` in git-only mode; the template then
renders paths as plain text. `risk.level` is 0, 1 or 2; `risk.label` is one of `grew`, `reworked`,
`hotspot`, `complex`, `large`; `risk.commits` is optional and rendered
verbatim. `line` is optional; GitLab needs both
`old` and `new` (see `hosts.md`). `status` is one of `added`, `modified`,
`deleted`, `renamed`.

## Phase 5: Publish

1. Copy `tour-template.html` to the scratchpad and replace the contents of
   the `tour-data` script block with the JSON. Change nothing else. The
   `<title>` is set by the page from `pr.title`, so the template's own title
   tag must be replaced with the PR title too (first 8KB is what the
   gallery reads).
2. Publish with the Artifact tool: favicon `🗺️`, description
   "Reading order for <repo>#<number>", and
   `capabilities: {db: {}, user: {}}` so each reviewer's ticks sync to
   their own private subtree of the artifact's database and follow them
   across devices (the page keeps `localStorage` as the instant copy and
   works without the grant). A `db` artifact is organization-internal:
   it cannot be shared by public link. When the tour must go to someone
   outside the organization, publish with `capabilities: {}` instead and
   say that ticks then stay in the browser. Do not load `artifact-design`
   for this; the page is already designed and the data block is the only
   variable part.
3. Report the link and one sentence on what the reviewer will find. Remind
   them once that links open the PR in a second tab that is reused.
4. With `--comment`, render the markdown form and post it (`hosts.md`).
   Keep it short: summary, then chapters as headings with a bullet per
   file (`path`, `why`), no `look_for`, no badges, and the artifact link at
   the top. End with a "Try it" list of the `qa.routes` paths and notes,
   without the base URL. Start the body with `<!-- jr-rails-pr-tour -->` so
   a re-run can find and edit it.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Every file is a singleton | grep ran against the working tree, not `$HEAD`, or the PR is all views/JS | Use `git show $HEAD:path`; for view-heavy PRs rely on the controller → view convention edges |
| One chapter holds 80% of the files | one god model touched everywhere | Split by layer (4a.5) and say in the summary that the PR is really one concern |
| `attractor: command not found` inside the project | the project's `.ruby-version` selects a Ruby without the gem | "Finding attractor": run it under another Ruby (`RBENV_VERSION=…`) or install into this one; never skip to the fallback |
| `attractor diff` prints nothing | old attractor without `diff` | `attractor version` ≥ 2.7; upgrade with `gem update attractor` under the same Ruby |
| `attractor diff` prints a header but no rows | `-c 0` missing (minimum churn 3 drops new files), or the language's plugin (`attractor-ruby`, `attractor-javascript`, `attractor-swift`, …) not installed | Add `-c 0`; `gem list attractor-` |
| Swift/Kotlin rows missing, `lizard is not installed` in stderr | the plugin runs lizard via `uvx`; `uv` is not on PATH | `brew install uv`, or `pip install lizard`, or `ATTRACTOR_LIZARD="python -m lizard"` |
| Links open a new tab every click | `rel="noopener"` got added, or the browser blocks popups for the artifact origin | Keep the template's anchor markup; allow popups once |
| Link lands on the PR but not the file | large diff not yet loaded by the host | Scroll once, then click again; GitHub loads diffs lazily |
| GitLab line anchor does nothing | missing `old` line | Link to the file instead |
