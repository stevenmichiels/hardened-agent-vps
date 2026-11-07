# Agent Workflows

Claude Code CLI and Codex CLI are interactive coding tools for Git repositories
and disposable workspaces. They should not write directly to production runtime
directories, production env files, or production databases. Move changes to
live systems through reviewed Git diffs, CI/deploy scripts, Ansible, or an
explicit staging promotion.

## Dual-Agent Review Pattern

Claude Code implements and Codex reviews. Do not let both agents edit the same
worktree at the same time.

The base role installs the `codex-review` Claude skill and the
`claude-codex-review` helper, which asks Codex CLI for a review through Codex's
native `codex review` command. The helper can include recent plan context and
writes a Markdown report. It does not give the reviewer edit tools, stage files,
or create commits.

## Ask Codex to Review Claude Work

```sh
claude-codex-review \
  "Review the current diff as a strict senior engineer."

claude-codex-review --commit HEAD -o codex-review.md

claude-codex-review --base main \
  "Review this branch against main."
```

## Review Artifacts

Default reports go under local review directories. If a report should be
versioned, inspect it first and commit only that Markdown file in a separate
commit.

The helper refuses likely secret-bearing paths and oversized diffs by default.
Override those protections only after manually reviewing what will be sent to
the second model. Plan context can also contain sensitive material and should
be disabled when it is irrelevant.

The source-controlled template lives in `templates/claude-skills/codex-review/`.
