# CLAUDE.md

Claude Code plugin with stock-screening and portfolio-analysis skills. Skills live in
`skills/<name>/SKILL.md`, agents in `agents/`, manifests in `.claude-plugin/`. Everything
here is prose that Claude reads at runtime — there is no build and no test suite.

## Publishing — read this before any push

Two remotes, and they are **not** interchangeable:

| Remote | Visibility | Branch | Contents |
| --- | --- | --- | --- |
| `origin` | private, internal git server | `main` | full development history |
| `github` | **public** | `main` ← local `publish` | squashed release commits only |

**Never push `main`, a task branch, or anything else carrying `main`'s history to
`github`.** The development history contains personal data. The public repo receives one
squashed commit per release and nothing else — no merges from `main`, no cherry-picked
ranges of internal commits, no `git push github main`.

`publish` is the only branch that may be pushed to `github`. Each of its commits is a
single squashed snapshot of `main`'s tree, so the public repo's history is a list of
releases, never a list of internal commits.

### Releasing

Bump the version in **both** `.claude-plugin/plugin.json` and
`.claude-plugin/marketplace.json` (they must match), commit that on `main`, then:

```bash
git fetch github
git checkout --detach github/main   # base on the remote, not the local publish ref
git read-tree -u --reset main       # take main's tree wholesale, including deletions
git diff --cached --stat            # review: this is exactly what goes public
git commit --no-verify -m "<one-line release summary>"
git diff --stat HEAD main           # must print nothing
git push github HEAD:main
git checkout -
```

`read-tree` + one `commit` is what keeps it to a single commit; do not `merge`,
`cherry-pick` or `rebase` from `main` onto `publish`.

Notes:

- Base the commit on `github/main`, not on local `publish`. `publish` is checked out in the
  primary worktree and goes stale whenever a release is pushed from a git worktree, which
  would turn the next push into a non-fast-forward.
- After pushing, advance the local ref with `git pull --ff-only` in the worktree that owns
  `publish`. Never move the `publish` ref from a worktree that does not own it — its files
  would read as a pending revert of the release.
- The pre-commit hook blocks every `git commit` without `--no-verify`; the version bump it
  asks about is the one described above.
- Keep private hostnames, internal repo paths, real portfolio files and account details out
  of every committed file — the whole tree goes public at the next release. `.gitignore`
  already excludes `*.xml`, `Shortlist/` and `Stock Screener/`.

## Editing skills

- A skill's `description` is its trigger. Changing it changes when the skill fires, so edit
  it deliberately and keep the existing trigger phrases.
- `quick-stock-analysis` always evaluates all five gates and reports a score out of five —
  it never short-circuits on a failing gate.
- `portfolio-management` installs the `pp` CLI (`go install
  github.com/from68/pp-cli/cmd/pp@latest`) at the start of every run that reads a Portfolio
  Performance file.
- Market data comes from the FMP and IBKR MCP connectors only. Never add a WebSearch or
  scraping fallback for a number that belongs in a metrics table.
- No `.mcp.json` ships with this plugin — connectors are configured per user in Claude.
