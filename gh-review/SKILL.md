---
name: gh-review
description: Use when asked to review a GitHub pull request - fetches PR into a worktree, runs code-reviewer and test-effectiveness-analyst agents, then posts per-line review comments back to the PR via gh CLI
---

<skill_overview>
Orchestrate GitHub PR code review: isolate in worktree, delegate to review agents, post per-line comments. Never auto-approves — always COMMENT.
</skill_overview>

<rigidity_level>
LOW FREEDOM - Never skip worktree isolation, agent delegation, or COMMENT-only posting.
</rigidity_level>

<quick_reference>
| Step | Action | Tool |
|------|--------|------|
| 1 | Gather PR metadata + diff | `gh pr view --json`, `gh pr diff` |
| 2 | Isolate in worktree (detached HEAD) | `git fetch origin pull/N/head` + `git worktree add --detach` (NOT `gh pr checkout`) |
| 3 | Run code-reviewer agent | Agent (subagent_type: code-reviewer) |
| 4 | Run test-effectiveness-analyst (if tests changed) | Agent (subagent_type: test-effectiveness-analyst) |
| 5 | Map findings to diff positions, post review | `gh api` with event=COMMENT |
| 6 | Cleanup | `git worktree remove` or `rm -rf` |
</quick_reference>

<when_to_use>
User asks to review a GitHub PR (URL, `owner/repo#N`, or PR number).
</when_to_use>

<the_process>

## 1. Gather PR context

```bash
gh pr view <PR> --json number,title,body,headRefName,baseRefName,files,author
gh pr diff <PR>
```

Parse the diff to build `{file: {line_number: diff_position}}`. The `position` is the line offset from the first `@@` hunk header per file, counting context + additions + deletions sequentially.

## 2. Isolate in worktree

**Never checkout in the user's working tree.**

In target repo — fetch the PR head and create a detached worktree (avoids branch conflicts):
```bash
git fetch origin pull/<N>/head:refs/pr/<N>
git worktree add /tmp/gh-review-<N> refs/pr/<N> --detach
```

Not in target repo:
```bash
git clone --depth=50 https://github.com/{owner}/{repo}.git /tmp/gh-review-<N>
cd /tmp/gh-review-<N> && git fetch origin pull/<N>/head && git checkout FETCH_HEAD --detach
```

**Do NOT use `gh pr checkout`** — it creates a named branch which fails if the branch is already checked out elsewhere. Always use detached HEAD via fetch + ref.

## 3. Delegate to agents

**Do NOT review code yourself.** Spawn `code-reviewer` agent with: PR title/description, full diff, changed file paths in worktree. Instruct it to report as: `file:line — [Critical|Important|Suggestion] — description`.

If test files changed (`*test*`, `*spec*`, `*_test.*`), also spawn `test-effectiveness-analyst` in parallel with the changed tests and the production code they cover.

## 4. Post review

Map each finding to a diff position. Findings outside the diff go in the review body.

**Build the ENTIRE payload as one JSON file.** Do NOT mix `-f` flags with `--input` — `gh api -f` passes values as strings, which corrupts the `comments` array and causes a 422 error.

```bash
# Write full payload to a temp file (Bash heredoc or Write tool)
cat > /tmp/gh-review-payload.json << 'PAYLOAD'
{
  "event": "COMMENT",
  "body": "## Automated Code Review\n\nSummary of findings...",
  "comments": [
    {"path": "src/file.go", "position": 12, "body": "**[Critical]** description"},
    {"path": "src/other.go", "position": 5, "body": "**[Important]** description"}
  ]
}
PAYLOAD

gh api repos/{owner}/{repo}/pulls/{number}/reviews \
  --method POST --input /tmp/gh-review-payload.json
```

**NEVER use APPROVE or REQUEST_CHANGES.** Always COMMENT.
**NEVER use `-f` flags for the review post** — only `--input` with the full JSON payload.

## 5. Cleanup

```bash
git worktree remove /tmp/gh-review-<N> --force  # or rm -rf for clones
```

</the_process>

<examples>
<example>
<scenario>Agent reviews code itself instead of delegating</scenario>
<why_it_fails>Misses SRE-level structured analysis. Ad-hoc criteria instead of established framework.</why_it_fails>
<correction>You are the orchestrator, not the reviewer. Spawn agents, collect findings, post them.</correction>
</example>

<example>
<scenario>Agent skips worktree because repo isn't cloned locally</scenario>
<why_it_fails>Agents lose file context — diff hunks alone miss surrounding code. "No local repo" becomes excuse to skip isolation.</why_it_fails>
<correction>Clone it: `git clone --depth=50 ... /tmp/gh-review-N`. Agents need full files.</correction>
</example>
</examples>

<critical_rules>
1. **NEVER review code yourself** — delegate to agents
2. **NEVER checkout in user's working tree** — always /tmp worktree or clone
3. **NEVER use `gh pr checkout`** — it creates a named branch that conflicts when the branch is already checked out. Use `git fetch origin pull/<N>/head` + detached HEAD instead
4. **NEVER use APPROVE or REQUEST_CHANGES** — always COMMENT
5. **ALWAYS cleanup** worktree/clone after posting
6. **ALWAYS map findings to diff positions** — wrong positions = comments on wrong lines
7. **NEVER use `gh api -f` for posting reviews** — `-f` passes values as strings, corrupting the `comments` array (HTTP 422). Write the full JSON to a file and use `--input` only

**Excuses that don't work:**
- "Small diff, I'll review myself" → Agents apply structured analysis you'll skip
- "No local repo, just use the diff" → Clone it. Agents need file context
- "Just a README change" → Agents catch broken links, factual errors, introduced typos
- "Approve since it looks clean" → COMMENT only. Humans decide approval
- "gh pr checkout is simpler" → It breaks when the branch exists elsewhere. Always fetch + detach
- "I'll use -f for the comments" → `-f` passes strings, not JSON. The comments array becomes a string literal and GitHub returns 422. Always `--input` with full JSON file
</critical_rules>

<integration>
**Agents:** `hyperpowers:code-reviewer`, `hyperpowers:test-effectiveness-analyst` (conditional)
**Requires:** `gh` CLI authenticated with repo access
</integration>
