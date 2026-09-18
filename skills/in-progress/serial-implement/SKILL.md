---
name: serial-implement
description: "Implement every sub-issue of a Linear parent issue, one fresh subagent at a time on one branch, with a fresh review after each."
disable-model-invocation: true
argument-hint: "A Linear parent issue identifier (e.g. COINE-40) or URL"
---

# Serial Implement

Take a Linear **Parent issue** and implement each of its **Sub-issues** in dependency order, strictly one at a time, each in a fresh **implementer** subagent, all on one branch. You, the main session, own the flow: every Linear state change, every review, every exchange with the user. The implementer only reads its ticket, writes code, and commits.

The goal is fewer interruptions, not autonomy. The user answers real decisions and never confirms routine steps.

Linear is reached through the Linear MCP tools: `get_issue`, `list_issues`, `list_issue_statuses`, `save_issue`, `save_comment`. Everything written to Linear (comments, follow-up issues) is in Italian; code, identifiers, and commit messages stay in English.

## 1. Preflight

Nothing touches git until every check below passes. Any failed check ends the run with a message naming what failed.

1. Resolve the argument (identifier or URL) to the Parent issue with `get_issue`. Keep its identifier, team, and `gitBranchName`.
2. List its Sub-issues with `list_issues` (`parentId` = the parent; fields `id`, `title`, `status`, `statusType`, `labels`, `createdAt`). No Sub-issues: stop.
3. For each Sub-issue, `list_issues` with `parentId` = that Sub-issue. Any result means nested Sub-issues: stop.
4. Resolve the `ready-for-agent` label string: the mapping in `docs/agents/triage-labels.md` when the repo has one, else the literal `ready-for-agent`.
5. Classify every Sub-issue:
   - `statusType` `completed` or `canceled`: **skipped**
   - anything else carrying the label: **queued**
   - anything else: **not ready**
   One or more not ready: stop and list them with state and labels.
6. Order the queue. `get_issue` with `includeRelations: true` on each queued Sub-issue; sort topologically on the "blocked by" edges, blockers first, creation order as tie-break. A blocker that is skipped counts as satisfied. A cycle: stop and name the Sub-issues in it.
7. `list_issue_statuses` for the team. `In Progress`, `In Review`, and `Done` must all exist by name. Any missing: stop.
8. `git status --porcelain` must be empty. Otherwise ask the user whether to stash, commit, or abort. Never discard their changes.

Done when the queue is ordered, every Sub-issue is classified, and the three states are resolved to ids.

## 2. Branch

The branch name is the Parent issue's `gitBranchName`.

- It exists (locally or on the remote): check it out. This is a **resume**. Commits already on it belong to earlier runs and are listed in the next implementer's brief.
- It does not: update the default branch (`git pull`), then create the branch from it.

## 3. Work the queue

One Sub-issue at a time, in order. Dispatch nothing in parallel. Between Sub-issues, ask the user nothing.

1. `save_issue`: state `In Progress`.
2. Record `START`, the current `HEAD` SHA.
3. Spawn a fresh implementer subagent with the brief in [IMPLEMENTER-BRIEF.md](IMPLEMENTER-BRIEF.md), filled in. Wait for it.
4. Act on its outcome:
   - **BLOCKED**: put the question to the user. Post question and answer as a comment on the Sub-issue. Send the answer to the same subagent (it keeps its context) and wait again.
   - **FAILED**: go to [Stopping](#stopping).
   - **DONE** with no commits between `START` and `HEAD`: treat as FAILED.
   - **DONE**: continue.
5. `save_issue`: state `In Review`. Run the [review](#4-review-one-sub-issue).
6. `save_issue`: state `Done`. Comment: branch, each commit as SHA and subject, one line on the review outcome, and any delivery-boundary decision with its local evidence paths. Labels stay as they are.

## 4. Review one Sub-issue

1. Write the Sub-issue (title, description, comments) to a scratch file: that file is the spec.
2. Call the Skill tool with "code-review": fixed point `START`, the scratch file as the spec. Its two reviewers are fresh subagents.
3. Classify findings. **Blocking**: every Spec-axis finding (missing, partial, or wrong requirement) and every breach of a documented repo standard. **Non-blocking**: smells and judgement calls. A blocking Spec finding is a **delivery-boundary finding** when its only completion requires pushing, opening or updating a pull request, or attaching an artifact to a remote pull request. This skill never performs those operations.
4. Set delivery-boundary findings aside. Send every other finding, blocking ones marked, to the same implementer subagent for one fix round (the fix-round message is in the brief file). It fixes, runs the tests, commits. When there are no other findings, skip the fix round.
5. The round had ordinary blocking findings: review again (steps 1 to 3). Two reviews per Sub-issue is the cap.
6. Ordinary blocking findings remain after the second review: [escalate](#5-escalation).
7. When the latest review has delivery-boundary findings and no ordinary blocking findings remain, use [delivery-boundary escalation](#delivery-boundary-escalation). Never send those findings to the implementer.
8. Otherwise the review passed.

### Delivery-boundary escalation

Show the delivery-boundary findings and every local evidence path returned by the implementer. Ask the user to choose:

- **Accept**: record the unmet external requirement and local evidence paths, then the Sub-issue goes to Done as it is.
- **Defer**: create a follow-up issue in the same team and project for the pull-request-owning step, `relatedTo` the Sub-issue, with the local evidence paths; the Sub-issue goes to Done and the run continues.
- **Stop**: the Sub-issue stays `In Review`; go to [Stopping](#stopping).

When the user does not choose, Stop. `Guide` is not offered because an implementer working only on the local branch cannot complete a remote pull request operation.

## 5. Escalation

Show the user the blocking findings still open and ask which of these they want:

- **Accept**: the Sub-issue goes to Done as it is.
- **Guide**: the user gives direction. One more fix round with that direction, then up to two more reviews. Post the direction as a comment on the Sub-issue.
- **Defer**: create a follow-up issue in the same team and project for the open findings, `relatedTo` the Sub-issue; the Sub-issue goes to Done; the run continues.
- **Stop**: the Sub-issue stays `In Review`; go to [Stopping](#stopping).

When the user does not choose, Stop.

## Stopping

The run stops at the first FAILED or at a Stop in escalation. The Sub-issue keeps the state it had, the branch and its commits stay exactly as they are, nothing is reset. Then write the [report](#report).

Rerunning the skill on the same Parent issue resumes: Done Sub-issues are skipped, the branch is reused, and the brief of a Sub-issue that already has commits lists them so the implementer continues instead of starting over.

## Report

At the end, complete or stopped, tell the user:

- the branch
- per Sub-issue: done, skipped, or failed; its commits; one line on its review
- every question answered during the run
- every delivery-boundary finding, the user's decision, and all local evidence paths
- the remaining queue, when stopped

The skill never pushes, never opens a PR, and never modifies the Parent issue. Suggest `/code-review main` on the whole branch before the PR.

## Why the main session runs the review

`implement` has the implementer call `code-review`, which spawns two fresh reviewer subagents. A subagent cannot spawn subagents, so an implementer subagent calling `code-review` would review its own work inside its own context. Running the review from the main session keeps the reviewers fresh. The fix still goes back to the implementer because it already holds the context to make it cheaply.
