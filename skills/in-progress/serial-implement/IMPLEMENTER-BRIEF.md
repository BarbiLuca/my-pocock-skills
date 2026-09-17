# Implementer brief

Fill in the placeholders and send this as the subagent's prompt. Context pointers over copies: the implementer reads its ticket from Linear itself, so later comments on it are seen.

## Dispatch

```
You are implementing Linear Sub-issue <ID> of Parent issue <PARENT-ID>, on branch <BRANCH>, in the repo at <REPO-PATH>.

Read the Sub-issue (description and every comment) with the Linear MCP tool `get_issue`; read the Parent issue for context. The project's CLAUDE.md, CONTEXT.md, and ADRs apply. Read Linear only: change nothing there.

Already implemented on this branch: <list of Sub-issue identifiers with their commits, or "nothing yet">.
Commits already on this branch for this Sub-issue: <list of SHA and subject, or "none">. When there are any, continue from them: do not redo their work.

Process:
- Call the Skill tool with "tdd" where possible, at pre-agreed seams.
- Run typechecking regularly, single test files regularly, and the full test suite once at the end. Take the commands from the project's CLAUDE.md or package configuration.
- Commit to the current branch as you go: small atomic commits, semantic prefix, the Sub-issue identifier in every message, e.g. `feat: add expiry field (<ID>)`. Leave at least one commit.
- The review happens after you return; do not run code-review.

Questions: return BLOCKED only for a decision that the Sub-issue, CONTEXT.md, the ADRs, and CLAUDE.md leave open and that changes the result. Procedure and permission questions are yours to settle. You will receive the answer in a follow-up message; continue from where you stopped.

Return exactly one of:
- DONE, then the list of commits (SHA and subject).
- BLOCKED, then the question and the minimum context needed to answer it.
- FAILED, then the reason: what you tried and what is still red.
```

## Fix round

Sent to the same subagent after a review:

```
Review findings for <ID> (diff since <START>), blocking ones marked [BLOCKING]:

<findings>

Fix all of them, run the tests, commit with the identifier. Return DONE with the new commits, or FAILED with the reason.
```

## Guided fix round

Sent after the user chose Guide in escalation:

```
Blocking findings still open for <ID>: <findings>
The user's direction: <direction>

Fix the findings following the direction, run the tests, commit with the identifier. Return DONE with the new commits, or FAILED with the reason.
```
