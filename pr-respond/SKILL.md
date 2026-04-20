---
name: pr-respond
description: Use when responding to GitHub PR review comments - applies Socratic questioning to understand each comment's deeper concern, generates multiple response options per comment, and lets the user choose or write their own before posting replies
---

<skill_overview>
Respond to PR review comments through Socratic analysis: question the reviewer's intent and your own assumptions before generating response options. Never jump to "I'll fix this" without first understanding what's really being asked.
</skill_overview>

<rigidity_level>
MEDIUM FREEDOM - Socratic analysis per comment is mandatory. User always chooses what gets posted.
</rigidity_level>

<quick_reference>
| Step | Action |
|------|--------|
| 1 | Fetch comments: `gh api repos/{o}/{r}/pulls/{n}/comments --paginate` |
| 2 | Socratic analysis (5 questions) per comment |
| 3 | Categorize: Must-address / Should-address / Optional / Challenge |
| 4 | Generate 3 options: Agree & Act / Explore & Discuss / Respectfully Decline |
| 5 | User picks via AskUserQuestion (or writes their own) |
| 6 | Post: `gh api repos/{o}/{r}/pulls/{n}/comments/{id}/replies -f body="..."` |
</quick_reference>

<when_to_use>
User says "respond to PR comments", "address review feedback", or has unresolved review comments.
</when_to_use>

<the_process>

## 1. Fetch and group comments

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate
```

Group by file and thread. Identify top-level vs. replies.

## 2. Socratic analysis per comment

**Do NOT write any response yet.** For each comment, answer:

1. **What is the reviewer literally asking?**
2. **What deeper concern might this reflect?** (Codebase health, past incident, convention?)
3. **What assumptions is the reviewer making?** (Correct? Partially? Wrong?)
4. **What assumptions am I making?** (Defensive? Assuming my approach is right?)
5. **Does this need to be addressed at all?** (In scope? Genuine issue? Reviewer confused?)

Present analysis to user before response options.

## 3. Categorize

- **Must-address**: Bug, security, correctness — requires code change + response
- **Should-address**: Valid improvement — respond with plan
- **Optional**: Style nit, preference — accept or decline with reasoning
- **Challenge**: Reviewer may be wrong or out of scope — respectful dialogue, not compliance

## 4. Generate 3 response options

**A — Agree & Act**: Accept feedback, state what you'll change
**B — Explore & Discuss**: Ask a genuine clarifying question or propose alternative
**C — Respectfully Decline**: Acknowledge the point, explain why you'd keep current approach

All responses must be specific to the actual code, non-defensive, and reference the reviewer's concern directly.

## 5. User chooses via AskUserQuestion

Present per comment: Socratic summary, category, 3 options. User picks, edits, or writes their own via "Other". **Never auto-post.**

## 6. Post responses

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments/{comment_id}/replies \
  --method POST -f body="<chosen response>"
```

</the_process>

<examples>
<example>
<scenario>Agent immediately says "I'll fix this" without analysis</scenario>
<why_it_fails>Skips understanding WHY the reviewer raised it. "Why aren't you using the middleware?" could mean "duplicated code" OR "security consistency concern" — the right response differs.</why_it_fails>
<correction>Run 5 Socratic questions first. Then present 3 options. Let user choose.</correction>
</example>

<example>
<scenario>Reviewer is hostile and technically wrong</scenario>
<why_it_fails>Agent either mirrors hostility or caves to incorrect feedback out of deference.</why_it_fails>
<correction>Socratic analysis finds valid concern underneath (if any). All 3 options stay professional. "Challenge" category signals reviewer may be wrong — Explore option asks genuine clarifying question to surface the real concern.</correction>
</example>
</examples>

<critical_rules>
1. **NEVER skip Socratic analysis** — 5 questions before any response
2. **NEVER auto-post** — user confirms every response via AskUserQuestion
3. **ALWAYS offer 3 options** — Agree, Explore, Decline for every comment
4. **NEVER be defensive** — even Decline must acknowledge the reviewer's point
5. **ALWAYS let user write their own** — AskUserQuestion "Other" covers this

**Excuses that don't work:**
- "Obviously right, just agree" → Still offer 3 options. User has context you lack.
- "Just a nit, skip analysis" → Nits can reflect deeper concerns. Run the 5 questions.
- "Time pressure, batch-post" → Never. User confirms each one.
- "Reviewer is wrong, just decline" → Still run analysis. Find the valid concern underneath.
</critical_rules>

<integration>
**Requires:** `gh` CLI, AskUserQuestion tool
**Pairs with:** `gh-review` (reviews PRs) — this skill handles the other side (responding)
</integration>
