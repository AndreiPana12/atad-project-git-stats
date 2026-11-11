# Git Repository Statistics

A web service that reads a Git repository and reports what happened in it: who
contributed, how much and when, how activity changed over time, and which files
keep being rewritten. It is meant for someone who has to understand an
unfamiliar repository quickly — a new team member, a reviewer, a teacher —
without reading `git log` by hand.

- **Student:** Pana Andrei, group DMWD
- **Project:** 10 — Git Repository Statistics
- **Language:** Go

## What it does

You give the service the path to a Git checkout. It walks the history once,
without modifying the repository, and then answers questions about it over
HTTP: per-contributor commits, lines added and removed, first and last
activity; commits per day, week or month; and the files changed most often.
Reports can be narrowed to a date range, a subdirectory or a file pattern, and
contributors who appear under several names or email addresses can be folded
into one person by configurable rules.

Merge commits are the main difficulty. A merge touches every line that came in
on its branch, so counting it like an ordinary commit credits the person who
merged with work somebody else wrote. The service therefore takes a merge
policy (`count` or `ignore`) for every analysis, and every contributor report
states which policy produced it. It does not judge people or code quality; it
only counts, and says how it counted.

**Current state (week 7):** only the skeleton is built. The service starts in a
container, answers the four common routes (`/health`, `/version`, `/`,
`POST /reset`) and is built and published by CI. None of the project-specific
routes exist yet. [`docs/architecture.md`](docs/architecture.md) describes the
planned design and lists what is still missing.

## How to run it

```
docker run --rm -p 8080:8080 ghcr.io/andreipana12/atad-project-git-stats:latest
```

To build and run without Docker (needs Go 1.27):

```
go run ./cmd/app
go run ./cmd/app --version
```

To run the tests:

```
go test ./...
```

## What you should see

The service logs one line and listens on port 8080:

```
level=INFO msg=listening addr=:8080 commit=...
```

A health check returns 200 and the time since start:

```
$ curl -i http://localhost:8080/health
HTTP/1.1 200 OK
Content-Type: application/json

{"status":"ok","uptime_seconds":3}
```

`--version` prints one line of JSON and exits 0:

```
$ go run ./cmd/app --version
{"app":"atad-project","version":"0.0.0-dev","commit":"unknown","built_at":""}
```

`commit` is `unknown` with `go run`. In the published image it is the SHA of
the commit CI built from, which is also what `GET /version` returns.

Opening <http://localhost:8080/> in a browser shows the student and project
details.