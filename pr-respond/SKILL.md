---
name: pr-respond
description: Use when responding to GitHub PR review comments - applies Socratic questioning to understand each comment's deeper concern, generates multiple response options per comment, ensures every comment is addressed with code changes where needed, and lets the user choose or write their own before posting replies
---

<skill_overview>
Respond to PR review comments through Socratic analysis: question the reviewer's intent and your own assumptions before generating response options. Never jump to "I'll fix this" without first understanding what's really being asked. Address EVERY comment -- none get skipped, forgotten, or batched away.
</skill_overview>

<rigidity_level>
MEDIUM FREEDOM - Socratic analysis per comment is mandatory. User always chooses what gets posted. Every comment must be addressed.
</rigidity_level>

<quick_reference>
| Step | Action |
|------|--------|
| 1 | Fetch ALL comments: `gh api repos/{o}/{r}/pulls/{n}/comments --paginate` |
| 2 | Build full inventory -- count comments, list each with file:line and snippet |
| 3 | Socratic analysis (5 questions) per comment |
| 4 | Categorize: Must-address / Should-address / Optional / Challenge |
| 5 | Generate 3 options: Agree & Act / Explore & Discuss / Respectfully Decline |
| 6 | User picks via AskUserQuestion (or writes their own) |
| 7 | Make code changes for accepted findings BEFORE posting responses |
| 8 | Commit and push all changes to the PR branch |
| 9 | Post replies with change summary: `gh api .../{id}/replies -f body="..."` |
| 10 | Verify: count responses posted == count comments received |
</quick_reference>

<when_to_use>
User says "respond to PR comments", "address review feedback", or has unresolved review comments.
</when_to_use>

<the_process>

## 1. Fetch and inventory ALL comments

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate
```

Group by file and thread. Identify top-level vs. replies.

**Build an explicit inventory before doing anything else:**
- Count total comments
- List each with: comment ID, file:line, severity tag (if present), first ~80 chars of body
- Present the inventory to the user: "Found N comments across M files. Here's the full list:"

This inventory is your checklist. Every entry must be addressed by the end. No exceptions.

## 2. Socratic analysis per comment

**Do NOT write any response yet.** For each comment, answer:

1. **What is the reviewer literally asking?**
2. **What deeper concern might this reflect?** (Codebase health, past incident, convention?)
3. **What assumptions is the reviewer making?** (Correct? Partially? Wrong?)
4. **What assumptions am I making?** (Defensive? Assuming my approach is right?)
5. **Does this need to be addressed at all?** (In scope? Genuine issue? Reviewer confused?)

Present analysis to user before response options.

## 3. Categorize

- **Must-address**: Bug, security, correctness -- requires code change + response
- **Should-address**: Valid improvement -- respond with plan
- **Optional**: Style nit, preference -- accept or decline with reasoning
- **Challenge**: Reviewer may be wrong or out of scope -- respectful dialogue, not compliance

## 4. Generate 3 response options

**A -- Agree & Act**: Accept feedback, state what you'll change. If chosen, the code change MUST happen before the response is posted.
**B -- Explore & Discuss**: Ask a genuine clarifying question or propose alternative
**C -- Respectfully Decline**: Acknowledge the point, explain why you'd keep current approach

All responses must be specific to the actual code, non-defensive, and reference the reviewer's concern directly.

## 5. User chooses via AskUserQuestion

Present per comment: Socratic summary, category, 3 options. User picks, edits, or writes their own via "Other". **Never auto-post.**

## 6. Make code changes BEFORE posting responses

For every comment where the user chose "Agree & Act":
1. Read the relevant file and understand the context
2. Make the code change
3. Run tests/build to verify the change doesn't break anything
4. Track what you changed: file path, what was changed, and why

**Do NOT post "I'll fix this" and leave the fix for later.** The code change, the commit, and the response are a unit -- ship all three or none.

## 7. Commit and push all changes to the PR branch

After ALL code changes are made and verified:

```bash
# Stage the specific files you changed
git add <file1> <file2> ...

# Commit with a message referencing the PR review
git commit -m "address PR review feedback

- <summary of change 1>
- <summary of change 2>

Co-Authored-By: Claude <noreply@anthropic.com>"

# Push to the PR branch
git push

# Capture the commit SHA and build the link
COMMIT_SHA=$(git rev-parse HEAD)
# Link format: https://github.com/{owner}/{repo}/commit/{sha}
```

**Verify the push succeeded.** If it fails (e.g., branch protection, merge conflict), stop and alert the user -- do NOT post responses claiming changes were made if the push failed.

**Do NOT post any responses until the push is confirmed.** The reviewer must be able to see the changes on the PR before reading your replies.

## 8. Post responses with change summaries and verify completeness

For "Agree & Act" responses, include a concrete summary of what was changed with a clickable link to the commit:

```
Fixed — changed `<what>` in `<file>` (<why>).

See [<short-sha>](https://github.com/{owner}/{repo}/commit/{full-sha}).
```

The commit link is mandatory. The reviewer must be able to click through to see the exact diff.

For "Explore & Discuss" and "Respectfully Decline" responses, post the user's chosen text as-is.

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments/{comment_id}/replies \
  --method POST -f body="<chosen response with change summary>"
```

After posting all responses, verify against the original inventory:
- Count responses posted
- Compare to total comments from Step 1
- If any comment was not addressed, flag it immediately: "Warning: comment {id} on {file}:{line} was not addressed"

**The skill is not complete until every comment has a posted response AND all code changes have been pushed.**

</the_process>

<examples>
<example>
<scenario>Agent immediately says "I'll fix this" without analysis</scenario>
<why_it_fails>Skips understanding WHY the reviewer raised it. "Why aren't you using the middleware?" could mean "duplicated code" OR "security consistency concern" -- the right response differs.</why_it_fails>
<correction>Run 5 Socratic questions first. Then present 3 options. Let user choose.</correction>
</example>

<example>
<scenario>Reviewer is hostile and technically wrong</scenario>
<why_it_fails>Agent either mirrors hostility or caves to incorrect feedback out of deference.</why_it_fails>
<correction>Socratic analysis finds valid concern underneath (if any). All 3 options stay professional. "Challenge" category signals reviewer may be wrong -- Explore option asks genuine clarifying question to surface the real concern.</correction>
</example>

<example>
<scenario>PR has 15 comments, agent addresses 8 and moves on</scenario>
<why_it_fails>7 comments get silently dropped. The reviewer sees partial engagement and assumes the rest were ignored or missed. Trust erodes. Unaddressed comments block the PR.</why_it_fails>
<correction>Build the inventory first (Step 1). Process every comment through the full pipeline. Verify at the end that responses posted == comments received. If any were skipped, flag before claiming done.</correction>
</example>

<example>
<scenario>Agent posts "Good point, I'll fix this" but never makes the code change</scenario>
<why_it_fails>The response promises a fix that never materializes. The reviewer approves based on the promise, the PR merges with the bug still in it.</why_it_fails>
<correction>Code changes happen BEFORE responses are posted. "Agree & Act" means act first, then respond with what was changed. Never post a promise -- post a receipt.</correction>
</example>

<example>
<scenario>Agent makes code changes and posts responses but never commits or pushes</scenario>
<why_it_fails>The changes exist only locally. The reviewer reads "Fixed in X" but sees no new commits on the PR. They assume nothing happened, trust erodes, and the review stalls. Even worse -- the local changes may be lost if the working directory is cleaned up.</why_it_fails>
<correction>After making all code changes: stage, commit, push to the PR branch. Verify the push succeeded. Only THEN post responses. Include the commit SHA in "Agree & Act" replies so the reviewer can click through to the exact change.</correction>
</example>

<example>
<scenario>Agent addresses the Critical findings but skips the Suggestions and Nits</scenario>
<why_it_fails>Suggestions and Nits are still comments that deserve acknowledgment. Ignoring them signals "I only care about blocking issues." Even a brief "Good catch, fixed" or "Intentional -- here's why" takes 10 seconds and shows respect for the reviewer's time.</why_it_fails>
<correction>Every comment gets a response, regardless of severity. Low-severity comments can get shorter responses, but they cannot be skipped.</correction>
</example>
</examples>

<critical_rules>
1. **NEVER skip Socratic analysis** -- 5 questions before any response
2. **NEVER auto-post** -- user confirms every response via AskUserQuestion
3. **ALWAYS offer 3 options** -- Agree, Explore, Decline for every comment
4. **NEVER be defensive** -- even Decline must acknowledge the reviewer's point
5. **ALWAYS let user write their own** -- AskUserQuestion "Other" covers this
6. **ALWAYS address EVERY comment** -- build inventory first, verify count at end. Zero comments may be skipped, regardless of severity
7. **ALWAYS make code changes BEFORE posting "Agree & Act" responses** -- the fix and the response ship together. Never post a promise without the receipt
8. **ALWAYS commit and push changes BEFORE posting responses** -- the reviewer must see the changes on the PR. If the push fails, do NOT post responses claiming fixes were made
9. **ALWAYS include a concrete change summary with a commit link in "Agree & Act" responses** -- file path, what changed, and a clickable `https://github.com/{owner}/{repo}/commit/{sha}` link. "Fixed!" with no detail is not a response
10. **ALWAYS verify completeness before claiming done** -- count responses posted vs comments received. If they don't match, you're not done

**Excuses that don't work:**
- "Obviously right, just agree" -> Still offer 3 options. User has context you lack.
- "Just a nit, skip analysis" -> Nits can reflect deeper concerns. Run the 5 questions.
- "Time pressure, batch-post" -> Never. User confirms each one.
- "Reviewer is wrong, just decline" -> Still run analysis. Find the valid concern underneath.
- "That comment was just a suggestion, doesn't need a reply" -> Every comment needs a reply. A reviewer who took time to write deserves acknowledgment.
- "I addressed the important ones, the rest are minor" -> Minor comments still need responses. Verify your count matches.
- "I'll fix it in a follow-up" -> If you chose Agree & Act, fix it now. If it genuinely belongs in a follow-up, say so explicitly in the response and get user approval.
- "I'll push later" -> No. Push before posting responses. The reviewer needs to see the changes.
- "The push failed but I'll post the response anyway" -> Stop. If the push failed, the reviewer can't see your changes. Fix the push first.
- "Fixed!" -> Not a response. State what file, what changed, and include a clickable commit link.
</critical_rules>

<integration>
**Requires:** `gh` CLI, AskUserQuestion tool
**Pairs with:** `gh-review` (reviews PRs) -- this skill handles the other side (responding)
</integration>
