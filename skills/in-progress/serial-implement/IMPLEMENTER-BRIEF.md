# Implementer brief

Fill in the placeholders and send this as the subagent's prompt. Spawn it with no inherited context (on Codex, `fork_turns: "none"`), on the implementer model chosen at the start of the run: this prompt is everything it knows. Context pointers over copies: the implementer reads its ticket from Linear itself, so later comments on it are seen.

## Dispatch

```
You are implementing Linear Sub-issue <ID> of Parent issue <PARENT-ID>, on branch <BRANCH>, in the repo at <REPO-PATH>.

Read the Sub-issue (description and every comment) with the Linear MCP tool `get_issue`; read the Parent issue for context. The project's CLAUDE.md, GLOSSARY.md, and ADRs apply. Read Linear only: change nothing there.

Already implemented on this branch: <list of Sub-issue identifiers with their commits, or "nothing yet">.
Commits already on this branch for this Sub-issue: <list of SHA and subject, or "none">. When there are any, continue from them: do not redo their work.
Review fixed point: <START>. Every commit for this Sub-issue after that point remains in review scope.

Process:
- Call the Skill tool with "tdd" where possible, at pre-agreed seams.
- Run typechecking regularly, single test files regularly, and the full test suite once at the end. Take the commands from the project's CLAUDE.md or package configuration.
- When files need changes, commit to the current branch as you go: small atomic commits, semantic prefix, and the Sub-issue identifier as a standalone token in every subject, e.g. `feat: add expiry field (<ID>)` or `fix: <ID> handle empty input`. A resume whose existing commits already satisfy the ticket needs no manufactured commit; verify the implementation and tests instead.
- Work only on the local branch: never push, open or update a pull request, or attach artifacts to one. When a requirement needs a pull request, produce any useful local evidence and retain its paths for the pull-request-owning step.
- The review happens after you return; do not run code-review.

Questions: return BLOCKED only for a decision that the Sub-issue, GLOSSARY.md, the ADRs, and CLAUDE.md leave open and that changes the result. Procedure and permission questions are yours to settle. You will receive the answer in a follow-up message; continue from where you stopped.

Return exactly one of:
- DONE, then the complete list of this Sub-issue's commits (SHA and subject), whether this dispatch created a commit, the verification commands with one line of result each, and every local evidence path needed by a requirement outside this skill's delivery boundary, or `none`.
- BLOCKED, then the question and the minimum context needed to answer it.
- FAILED, then the reason: what you tried and what is still red.
```

## Fix round

Sent to the same subagent after a review:

```
Ordinary review findings for <ID> (diff since <START>), blocking ones marked [BLOCKING]:

<findings>

Fix all of them, run the tests, commit with the identifier. Return DONE with the new commits, or FAILED with the reason.
```

## Guided fix round

Sent after the user chose Guide in escalation:

```
Ordinary blocking findings still open for <ID>: <ordinary-findings>
The user's direction: <direction>

Fix the findings following the direction, run the tests, commit with the identifier. Return DONE with the new commits, or FAILED with the reason.
```
