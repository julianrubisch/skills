---
name: jr-rails-pr-tour
description: >-
  Turn a large Rails pull/merge request into a guided reading order: chapters
  of related files, ordered by Rails layer and dependency, each with a
  one-line "why here" and a churn-x-complexity risk badge. Published as an
  artifact that drives the PR in a second browser tab. GitHub and GitLab.
triggers:
  - /jr-rails-pr-tour
  - jr-rails-pr-tour
  - where do I start reading this PR
  - reading order for this pull request
  - reading order for this merge request
  - guide me through this PR
  - tour this PR
  - this PR is huge, where do I start
  - order the files in this PR
  - review map for this PR
---

# Rails PR Tour

Code review tools list files alphabetically. For a 40-file PR that is the
wrong order: reviewers report more context switching and missed bugs, and what
they ask for instead is dependency order and grouping by purpose (Rahman et
al., ICSE 2026). Rails makes both cheap: the layer is in the path, and the
dependencies are in the constants.

This skill produces a **tour**: chapters of related files in the order a
reviewer should read them, with a reason per file and a risk marker where the
reviewer should slow down. It is a map, not a review. A review skill
(`/code-review`, `jr-rails-second-opinion`, or your own) does the
reviewing; this skill tells you where to look first.

Read `@references/guide.md` and follow it. Do not proceed without it.
Host commands and deep-link anchors are in `@references/hosts.md`. The
artifact page is `@references/tour-template.html`; fill its data block, do
not rewrite the page.

## Invocation

- `/jr-rails-pr-tour <PR or MR URL>`
- `/jr-rails-pr-tour <number>` (host inferred from `git remote`)
- `/jr-rails-pr-tour` (current branch vs `main`, no host links)
- `--comment` also posts the tour as a PR/MR comment for co-reviewers

## Operator

Run by the **reviewer**, from a checkout of the repository under review. The
consumer is a **human** reading the PR in the browser. The artifact opens in
one tab and every link targets a second, named tab holding the PR, so the
tour drives the diff. GitHub and GitLab refuse to be framed
(`X-Frame-Options`), so an embedded diff is not an option; named-window
links are the substitute.

## Phase summary

| Phase | What | Output |
|-------|------|--------|
| 0 | Resolve host, PR, base and head; fetch the diff | `HOST`, `PR`, file list with stats |
| 1 | Noise filter: lockfiles, schema dump, fixtures, generated assets | skip list |
| 2 | Signals: Rails layer per file, symbol references between changed files, commit structure | edge list, layer map |
| 3 | Risk: complexity the PR added (`attractor diff`, else `flog`, else diff size) and how many of the PR's commits touched each file; **ask** whether to add the churn × complexity scatter | badge per file, optional chart data |
| 4 | Chapters: cluster by coupling, order chapters by layer, order files by dependency; **name and reshape chapters by judgement**; list the routes to open on a dev server per chapter | tour JSON |
| 5 | Publish the artifact; optionally post the markdown comment | link |

## Hard rules

**Read-only on the repository.** The skill inspects the checkout and may
create temporary worktrees (attractor does this itself); it never edits,
commits, or checks out a different ref in the user's working tree.
Installing the attractor gems into the user's Ruby is allowed and expected
when no installed Ruby has them (guide, "Finding attractor"); say so in
one line when you do it. Starting the project's own devcontainer for QA is
allowed, and so is a throwaway one generated in the scratchpad and passed
with `--config` (`references/qa-devcontainer.md`); starting a server on
the host, or adding a devcontainer to the repository, is not.

**Every file in the diff appears exactly once**, in a chapter or in the skip
list. A tour that silently drops files defeats its purpose.

**Chapters are a narrative, not a bucket sort.** Phase 4 computes clusters
mechanically and then you name them and, where the mechanics produced
nonsense, reshape them. Say in the chapter's lede what the chapter is about.
"Miscellaneous" is not a chapter name.

**Risk badges annotate, they never reorder.** The reading order comes from
dependency and layer. A badge tells the reviewer where to slow down, not to
jump ahead. Badges measure the *change* (complexity added, commits in the
PR that touched the file), not the code's history; a historical hotspot is
only a modifier.

**Write in Simplified Technical English.** One idea per sentence, active
voice, the code's own names, no metaphors. The rules are in the guide's
"Writing rules" section; the reader is scanning between two windows.

**Do not review.** No findings, no severity, no suggestions. If you notice a
bug while building the tour, put one neutral sentence in the file's "what to
look for" line and move on.

## File layout

```
jr-rails-pr-tour/
├── SKILL.md                     # This file (entry point)
└── references/
    ├── guide.md                 # Phases 0-5 in detail, tour JSON schema
    ├── hosts.md                 # gh / glab commands, anchor formats, comment posting
    ├── qa-devcontainer.md       # Throwaway container for QA, outside the repo (load only on a yes)
    └── tour-template.html       # Artifact page; fill the data block only
```

Load reference files only when you need them.
