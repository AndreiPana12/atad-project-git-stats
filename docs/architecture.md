# Architecture note — Git Repository Statistics

Pana Andrei, group DMWD · Project 10 · week 7

Little is built yet (see the last section), so this note mostly describes the
design I plan to build.

## The shape

One Go binary serving HTTP on port 8080. `cmd/app` holds the routes; the work
goes in `internal/gitlog` (runs `git`, parses its output into commit records),
`internal/store` (the analyses, keyed by integer id) and `internal/report`
(filters, identity folding, aggregation).

1. `POST /repositories` with `{"path","merge_policy"}` is validated (400 if
   bad), gets the next id, is stored as `analysing` and returns 202 at once.
2. A goroutine runs one `git log` over the repository, parses the stream and
   marks the analysis `done` (or `failed`) with `analysed_in_ms`.
3. `GET /repositories/{id}/contributors`, `/activity` or `/churn` reads the
   stored commits, applies `from`, `to` and `path`, folds identities,
   aggregates and returns JSON: 404 for an unknown id, 409 while analysing.

In Docker the repository is mounted read-only (`-v …:/repos/sample:ro`), so it
cannot be modified even by mistake.

## The data

Per analysis: id, path, merge policy, state, timing and the list of commits.
Per commit: hash, author name and email, time, whether it is a merge, and for
each file the path, lines added and lines removed. The rules from
`PUT /identities` are kept separately. Everything is in memory, in a map behind
a mutex. Nothing survives a restart; `POST /reset` empties the map and the
rules. Reports are computed per request and not stored.

## Three decisions

**1. One `git log --numstat` process, read as a stream.** I rejected one
process per commit, which the project warns will not survive a large history.
I also rejected go-git: it would avoid needing `git` in the image, but it must
diff trees itself to count lines, which I expect to be slower. That expectation
is not measured yet. The cost of my choice is that `git` must be in the image.

**2. What the merge policies mean.** `ignore` drops merge commits
(`--no-merges`), so people are credited only for commits they made. `count`
keeps each merge and credits its diff against the first parent
(`--diff-merges=first-parent`) to whoever merged. On a test repository this
credited a 50-line feature to both its author and the merger, so total lines
exceed the real change. I chose it because it makes the difference visible. The
alternative, merges counted with zero lines, would make the two reports nearly
identical and hide the problem. Every contributor response states its policy.

**3. Store raw commits and aggregate per request.** Storing finished totals
would be smaller and faster, but `from`, `to`, `path` and identity rules all
change the totals, so each new filter would mean running `git` again. With raw
records, a new `PUT /identities` applies to analyses already done. I rejected
SQLite for the same reason: nothing has to survive a restart.

## What is not built yet

Built: the template's four common routes, `--version`, the Dockerfile, CI that
tests and publishes the image, and my details on `/`. Nothing else.

Not built: every project route; the `git log` parser; both merge policies;
identity rules (I have not chosen a rule format); the `from`/`to`/`path`
filters; grouping by day, week or month; a human-readable report besides JSON
(form not decided); analysing a `{"url"}`, which needs a clone into a temporary
folder that I have not designed. `POST /reset` clears nothing yet.

Not checked: whether 10,000 commits finish in reasonable time and memory. I
have no timing yet. `git` is not in the image, and will likely refuse a mounted
repository owned by another user ("dubious ownership") until `safe.directory`
is set. No tests of my own yet; `TestYourLogic` is still skipped.