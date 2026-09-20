# Hosts: GitHub and GitLab

Everything host-specific lives here. The guide refers to `HOST` (`github` or
`gitlab`), `PR` (number), `PR_URL`, `BASE` and `HEAD` (refs).

## Detecting the host

From a URL: `github.com/…/pull/N` or `gitlab.com/…/-/merge_requests/N`
(self-hosted GitLab has the same `/-/merge_requests/` path). From a bare
number, read the origin:

```bash
git remote get-url origin
# git@github.com:owner/repo.git      -> github
# git@gitlab.com:group/sub/repo.git  -> gitlab (groups can nest)
# git@git.example.com:group/repo.git -> ask glab (self-hosted GitLab)
```

Any other host is probably a self-hosted GitLab or GitHub Enterprise. Ask
the CLIs before asking the user: `glab auth status` lists every GitLab host
it is logged in to, and `gh auth status` does the same for GitHub. If the
remote's host appears in one of them, that is the answer. Only when neither
knows it: ask, and tell the user which login command to run
(`glab auth login --hostname git.example.com`).

Private repositories need no special handling: the CLIs are authenticated,
and the artifact only holds links that the reviewer's own browser session
follows. Nothing from the PR is fetched by the page itself.

Both CLIs must be authenticated: `gh auth status`, `glab auth status`. If
one is missing or logged out, fall back to the git-only path in the guide
(no deep links, no comment) and say so.

## Fetching the PR

| | GitHub | GitLab |
|---|---|---|
| Metadata | `gh pr view $PR --json title,body,baseRefName,headRefName,headRefOid,baseRefOid,url,additions,deletions,changedFiles` | `glab mr view $PR --output json` (fields: `title`, `description`, `source_branch`, `target_branch`, `sha`, `diff_refs.base_sha`, `web_url`) |
| Files with stats | `gh pr view $PR --json files --jq '.files[] \| [.path,.additions,.deletions] \| @tsv'` | `glab mr diff $PR` and count from the unified diff, or `glab api projects/:id/merge_requests/$PR/diffs` |
| Full diff | `gh pr diff $PR` | `glab mr diff $PR` |
| Commits | `gh pr view $PR --json commits --jq '.commits[] \| [.oid,.messageHeadline] \| @tsv'` | `glab api projects/:id/merge_requests/$PR/commits --paginate` |

Make sure the head and base commits exist locally before any per-file work:
`git fetch origin $BASE $HEAD` (for GitHub PRs from forks,
`gh pr checkout $PR --detach` into a temporary worktree is simpler; do not
check out in the user's working tree).

`gh` on GitHub Enterprise and `glab` on self-hosted GitLab work the same way
once the host is logged in (`gh auth login --hostname`, `glab auth login
--hostname`); inside the checkout both CLIs pick the host from the remote,
so `GH_HOST` / `GITLAB_HOST` are only needed when running outside it. The
diff anchors below are the same on every instance: they are computed by
GitLab and GitHub code, not by the hosted service.

## Deep links into the diff

Both hosts anchor files in the diff view by a hash of the file path, so
links are computed offline, no API call needed. The template computes these
in the browser with `crypto.subtle`; you only supply `path` and, optionally,
line numbers.

| | GitHub | GitLab |
|---|---|---|
| Diff page | `$PR_URL/files` | `$PR_URL/diffs` |
| File anchor | `#diff-` + sha256(path) as hex | `#` + sha1(path) as hex |
| Line anchor | file anchor + `R<new_line>` (or `L<old_line>`) | file anchor + `_<old_line>_<new_line>` |

Examples for `app/models/user.rb`:

```
https://github.com/o/r/pull/12/files#diff-c0d8…R42
https://gitlab.com/g/r/-/merge_requests/12/diffs#2fa1…_40_42
```

Shell equivalents, for a spot check before publishing:

```bash
printf %s app/models/user.rb | shasum -a 256 | cut -c1-64   # github
printf %s app/models/user.rb | shasum -a 1   | cut -c1-40   # gitlab
```

Spot-check one link per tour by opening it: GitHub's diff view lazy-loads
large diffs and a file that has not rendered yet will not scroll into view
until it loads. A tour that links to a file the host collapsed ("Load diff")
still lands on the right file header, which is enough.

GitLab needs both line numbers for a line anchor. For an added line the old
number is the line before the insertion on the old side; if you do not have
it, link to the file, not the line.

## Named-window links

Every link in the artifact uses `target="pr"`. The first click opens a second
tab; each later click navigates that same tab. That gives the reviewer the
two-window setup (tour left, diff right) without frames. The template does
this; do not change the target name, and do not add `rel="noopener"` (it
would sever the window name and spawn new tabs).

## Posting the tour as a comment (`--comment`)

The template's data block renders to markdown with the same chapter order
(see the guide's markdown section). Post it with:

```bash
gh pr comment $PR --body-file tour.md
glab mr note $PR --message "$(cat tour.md)"
```

Post once. If a tour comment already exists (search for the
`<!-- jr-rails-pr-tour -->` marker), edit it instead:

```bash
gh api repos/:owner/:repo/issues/comments/$ID -X PATCH -f body=@tour.md
glab api projects/:id/merge_requests/$PR/notes/$ID -X PUT -f body=@tour.md
```
