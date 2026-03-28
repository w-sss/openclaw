# Bug Fix Workflow — Lessons from #54556

**Date:** 2026-03-28  
**Incident:** PR #54569 duplicated existing correct fix #54566

## The Mistake

Submitted PR #54569 to fix issue #54556 (Matrix media `__dirname` error), but:
- Did not check for existing PRs first
- Misdiagnosed the root cause (thought it was `music-metadata`, actually `crypto-nodejs`)
- Wasted time on wrong fix while @joelnishanth had already opened correct PR #54566

## Correct Workflow

Before submitting any bug fix:

### 1. Search Existing Issues and PRs

```bash
# Check if issue already has linked PRs
gh issue view <ISSUE_NUMBER> --json comments,reactionGroups

# List PRs that reference this issue
gh pr list --search "<ISSUE_NUMBER>" --state open

# Check PR history by issue number in commit messages
git log --all --oneline --grep="<ISSUE_NUMBER>" | head -10
```

### 2. Review Existing PRs First

If a PR already exists:
- Read the diff: `gh pr view <PR_NUMBER> --json files`
- Check if it addresses the reported root cause
- Review comments for maintainer feedback
- **Only open a new PR if the existing one is wrong or incomplete**

### 3. Trace the Full Code Path

For media send/receive bugs:
- ✅ Sending path: `extensions/matrix/src/matrix/send/media.ts`
- ✅ Receiving path: `extensions/matrix/src/matrix/monitor/media.ts`
- ✅ Error location: Find exact stack trace source

**Example from #54556:**
- Symptom: `__dirname is not defined in ES module scope`
- My mistake: Assumed it was `music-metadata` (used in duration parsing)
- Reality: `@matrix-org/matrix-sdk-crypto-nodejs` uses `__dirname` to locate native `.node` bindings
- Correct fix: Load via `createRequire(import.meta.url)` (PR #54566)

### 4. Verify Root Cause Matches Error

Questions to ask:
- Does the error stack trace point to my suspected module?
- Is the affected code path actually used for the reported scenario?
- Does my fix touch the failing code path?

**Example from #54556:**
- ❌ My fix: `resolveMediaDurationMs()` — only called for audio/video
- ❌ Reported bug: Image send/receive failure
- ❌ My fix doesn't touch receive path at all
- ✅ Correct fix: `crypto-node.runtime.ts` — used by all E2EE media operations

### 5. Decision Tree

```
Issue reported
    ↓
Search existing PRs
    ↓
┌─────────────────┬─────────────────┬─────────────────┐
│ No PR exists    │ PR exists &     │ PR exists &     │
│                 │ correct         │ incorrect       │
└─────────────────┴─────────────────┴─────────────────┘
    ↓                 ↓                   ↓
Open PR           Review & approve    Open new PR with
                  Close duplicate     explanation why
                                      existing is wrong
```

## Commands Reference

```bash
# View issue with comments
gh issue view 54556 --json title,body,comments

# View PR diff
gh pr view 54566 --json files

# Check if PR is merged
gh pr view 54566 --json state,mergedAt

# Search commits by issue number
git log --all --oneline --grep="54556"

# List open PRs for repo
gh pr list --state open --limit 20
```

## Key Takeaways

1. **PR numbers are sequential** — if issue is #54556 and your PR is #54569, check #54557-#54568 first
2. **Maintainers see all PRs** — they will notice duplicate work and question your diligence
3. **Error messages have origins** — trace stack traces to the actual source module
4. **Code paths matter** — if bug affects images but your fix only touches audio/video, you're wrong
5. **Review before rushing** — 5 minutes checking existing PRs saves hours of wasted work

## Related Files

- `SOUL.md` — Core principles including this lesson
- `AGENTS.md` — Repository guidelines and contribution workflow
