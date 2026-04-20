---
name: gh-review
description: Use when asked to review a GitHub pull request - fetches PR into a worktree, runs exhaustive code-reviewer and test-effectiveness-analyst agents, then posts ALL findings as per-line review comments back to the PR via gh CLI
---

<skill_overview>
Orchestrate GitHub PR code review: isolate in worktree, delegate to review agents with exhaustive prompts, post ALL findings as per-line comments. Never auto-approves -- always COMMENT.
</skill_overview>

<rigidity_level>
LOW FREEDOM - Never skip worktree isolation, agent delegation, exhaustive prompting, or COMMENT-only posting.
</rigidity_level>

<quick_reference>
| Step | Action | Tool |
|------|--------|------|
| 1 | Gather PR metadata + diff + existing comments | `gh pr view`, `gh pr diff`, `gh api .../comments` |
| 2 | Isolate in worktree (detached HEAD) | `git fetch origin pull/N/head` + `git worktree add --detach` (NOT `gh pr checkout`) |
| 3 | Run code-reviewer agent (exhaustive) | Agent (subagent_type: code-reviewer) |
| 4 | Run test-effectiveness-analyst (if tests changed, exhaustive) | Agent (subagent_type: test-effectiveness-analyst) |
| 5 | Map ALL findings to diff positions, post review | `gh api` with event=COMMENT |
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

Also fetch existing review comments and reviews:
```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate
gh api repos/{owner}/{repo}/pulls/{number}/reviews --paginate
```

Build an **existing comments map**: `{file: {line: [comments]}}`. For each existing comment thread, classify it:
- **Open comment (no reply/resolution)** -- this line is already under review. Do NOT post a duplicate.
- **Acknowledged fix** ("fixed", "done", "updated", "addressed") -- verify the current code actually reflects the fix. If the fix looks correct, skip. If the claimed fix is wrong or incomplete, post a NEW comment: "Previous review flagged [issue]. The response indicated it was fixed, but [what's still wrong]."
- **Declined with reasoning** -- the author pushed back and gave a reason. Do not re-raise the same issue unless the reasoning is clearly incorrect.

## 2. Isolate in worktree

**Never checkout in the user's working tree.**

In target repo -- fetch the PR head and create a detached worktree (avoids branch conflicts):
```bash
git fetch origin pull/<N>/head:refs/pr/<N>
git worktree add /tmp/gh-review-<N> refs/pr/<N> --detach
```

Not in target repo:
```bash
git clone --depth=50 https://github.com/{owner}/{repo}.git /tmp/gh-review-<N>
cd /tmp/gh-review-<N> && git fetch origin pull/<N>/head && git checkout FETCH_HEAD --detach
```

**Do NOT use `gh pr checkout`** -- it creates a named branch which fails if the branch is already checked out elsewhere. Always use detached HEAD via fetch + ref.

## 3. Delegate to agents -- EXHAUSTIVE PROMPTS

**Do NOT review code yourself.** Spawn agents with exhaustive prompts that demand thoroughness.

### Code-reviewer agent prompt template

The prompt MUST include ALL of the following elements:

1. **Explicit exhaustiveness demand:** "Find EVERY issue, no matter how small. Be exhaustive. Do not stop at 5-10 findings. Go file by file through every changed file. I want a comprehensive list -- aim for 30+ findings."
2. **Full file list with worktree paths:** List every changed file as an absolute path in the worktree. Say "read each one fully from the worktree."
3. **The diff file path:** So the agent knows what changed vs what's existing code.
4. **Comprehensive review criteria checklist:** Include ALL of these:
   - Correctness of code logic
   - Memory safety (null dereference, use-after-free, resource leaks)
   - API design consistency with existing patterns in each file
   - Missing error handling at system boundaries
   - Potential runtime failures or crashes
   - Logical bugs (wrong variable, incorrect conditions, off-by-one)
   - Thread safety issues
   - Inconsistencies between similar methods (one overload checks something another doesn't)
   - Documentation/comment accuracy (swapped docs, stale comments)
   - Missing null checks on return values
   - Test quality issues (weak assertions, missing coverage, tautological tests)
   - Build/config issues
5. **Report format:** `file:line -- [Critical|Important|Suggestion|Nit] -- description`
6. **PR context:** Title, description, what the project is, key patterns.
7. **Existing comment threads:** Pass the existing comments map so the agent knows what's already been flagged. Tell it: "These lines already have review comments. Do NOT re-raise the same issues. If a comment says 'fixed' or 'addressed', verify the current code actually has the fix -- report only if the fix is missing or wrong."

**Do NOT use vague instructions like "focus on correctness and safety."** The checklist drives thoroughness -- vague prompts produce shallow reviews.

### Test-effectiveness-analyst agent prompt (if tests changed)

Spawn in parallel with the code-reviewer. The prompt MUST:
1. List every changed test file AND the production code it covers (both as worktree paths)
2. Demand exhaustive analysis: "Find every weak test, missing corner case, and coverage gap"
3. Ask for structured output per test: what it tests, why it's RED/YELLOW/GREEN, what to fix

## 4. Post review -- ALL findings

**Post EVERY new finding from BOTH agents as an inline comment.** Do not curate, filter, or drop findings — but DO deduplicate against existing comments:

- **Already commented (open, no resolution):** Skip — don't pile on.
- **Acknowledged as fixed:** Only post if the agent verified the fix is wrong/incomplete. Prefix with: "Previous review flagged this. The response said it was fixed, but..."
- **New finding on uncommented line:** Post normally.

### Position mapping

Map each finding's `file:line` to a diff position. Use a script to build the position map from the diff file.

- Findings that map to a diff position -> inline `comments` array
- Findings that do NOT map to a diff position (existing code, or lines outside the diff) -> append to the review `body` under a "Findings outside diff" section

### Posting

**Build the ENTIRE payload as one JSON file.** Do NOT mix `-f` flags with `--input` -- `gh api -f` passes values as strings, which corrupts the `comments` array and causes a 422 error.

```bash
# Use python for proper JSON escaping -- heredocs break on special characters
python3 -c "
import json
payload = {
    'event': 'COMMENT',
    'body': review_body_string,
    'comments': [
        {'path': 'src/file.go', 'position': 12, 'body': '**[Critical]** description'},
        ...
    ]
}
with open('/tmp/gh-review-payload.json', 'w') as f:
    json.dump(payload, f)
"

gh api repos/{owner}/{repo}/pulls/{number}/reviews \
  --method POST --input /tmp/gh-review-payload.json
```

**NEVER use APPROVE or REQUEST_CHANGES.** Always COMMENT.
**NEVER use `-f` flags for the review post** -- only `--input` with the full JSON payload.

## 5. Cleanup

```bash
git worktree remove /tmp/gh-review-<N> --force  # or rm -rf for clones
git update-ref -d refs/pr/<N>  # clean up the fetched ref
rm -f /tmp/gh-review-*.json /tmp/gh-review-*-diff.txt  # clean up temp files
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
<why_it_fails>Agents lose file context -- diff hunks alone miss surrounding code. "No local repo" becomes excuse to skip isolation.</why_it_fails>
<correction>Clone it: `git clone --depth=50 ... /tmp/gh-review-N`. Agents need full files.</correction>
</example>

<example>
<scenario>Agent prompt says "focus on correctness and safety" with no checklist</scenario>
<why_it_fails>Vague prompts produce 5-10 findings. The agent stops after finding the obvious issues and never checks docs, build config, test quality, API consistency, or cross-method inconsistencies.</why_it_fails>
<correction>Use the full 12-point review criteria checklist. Explicitly demand 30+ findings and say "go file by file." The checklist forces the agent to check each dimension for each file rather than stopping when it has "enough."</correction>
</example>

<example>
<scenario>Agent posts duplicate comments on lines that already have review feedback</scenario>
<why_it_fails>The PR already has 10 review comments from a prior round. The agent ignores them and posts the same issues again. The PR thread becomes noisy and the reviewer loses trust in the automated review.</why_it_fails>
<correction>Fetch existing comments in Step 1. Build a map of already-commented lines. Skip lines with open comments. For "acknowledged as fixed" threads, verify the fix in the current code -- only re-raise if the fix is wrong or incomplete.</correction>
</example>

<example>
<scenario>Orchestrator curates findings before posting -- drops "minor" ones</scenario>
<why_it_fails>Swapped XML docs, weakened test assertions, hardcoded magic numbers, and stale comments are all real issues that get silently dropped. The human reviewer never sees them.</why_it_fails>
<correction>Post EVERY finding. Let the severity tags (Critical/Important/Suggestion/Nit) communicate priority. The human decides what to act on.</correction>
</example>
</examples>

<critical_rules>
1. **NEVER review code yourself** -- delegate to agents
2. **NEVER checkout in user's working tree** -- always /tmp worktree or clone
3. **NEVER use `gh pr checkout`** -- it creates a named branch that conflicts when the branch is already checked out. Use `git fetch origin pull/<N>/head` + detached HEAD instead
4. **NEVER use APPROVE or REQUEST_CHANGES** -- always COMMENT
5. **ALWAYS cleanup** worktree/clone after posting
6. **ALWAYS map findings to diff positions** -- wrong positions = comments on wrong lines
7. **NEVER use `gh api -f` for posting reviews** -- `-f` passes values as strings, corrupting the `comments` array (HTTP 422). Write the full JSON to a file and use `--input` only
8. **ALWAYS use exhaustive agent prompts** -- include the full 12-point review criteria checklist, demand 30+ findings, say "go file by file through every changed file," and list every file as an absolute worktree path with "read each one fully"
9. **ALWAYS post ALL findings** -- never curate, filter, or drop agent findings. Findings outside the diff go in the review body. Severity tags communicate priority, not your editorial judgment

**Excuses that don't work:**
- "Small diff, I'll review myself" -> Agents apply structured analysis you'll skip
- "No local repo, just use the diff" -> Clone it. Agents need file context
- "Just a README change" -> Agents catch broken links, factual errors, introduced typos
- "Approve since it looks clean" -> COMMENT only. Humans decide approval
- "gh pr checkout is simpler" -> It breaks when the branch exists elsewhere. Always fetch + detach
- "I'll use -f for the comments" -> `-f` passes strings, not JSON. The comments array becomes a string literal and GitHub returns 422. Always `--input` with full JSON file
- "Too many findings, I'll post just the important ones" -> Post everything. The severity tags exist for a reason. Dropping findings means the reviewer misses real issues
- "The agent only found 6 things, that's enough" -> Your prompt was too vague. Use the full checklist and demand exhaustiveness. A 2500-line diff should produce 20-30 findings minimum
</critical_rules>

<integration>
**Agents:** `hyperpowers:code-reviewer`, `hyperpowers:test-effectiveness-analyst` (conditional)
**Requires:** `gh` CLI authenticated with repo access
</integration>
