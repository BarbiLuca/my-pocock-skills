---
name: serial-implement
description: "Implement every sub-issue of a Linear parent issue, one fresh subagent at a time on one branch, with a fresh review after each."
disable-model-invocation: true
argument-hint: "A Linear parent issue identifier (e.g. COINE-40) or URL"
---

# Serial Implement

Take a Linear **Parent issue** and implement each of its **Sub-issues** in dependency order, strictly one at a time, each in a fresh **implementer** subagent, all on one branch. You, the main session, own the flow: every Linear state change, every review, every exchange with the user. The implementer only reads its ticket, writes code, and commits.

The goal is fewer interruptions, not autonomy. The user answers real decisions and never confirms routine steps.

Linear is reached through the Linear MCP tools: `get_issue`, `list_issues`, `list_comments`, `list_issue_statuses`, `save_issue`, `save_comment`. Everything written to Linear (comments, follow-up issues) is in Italian; code, identifiers, and commit messages stay in English.

## Context budget

Every turn resends the whole conversation, so a run costs turns times context size. The rules below bound both and apply to the main session throughout this skill.

- **Orientation read**, one whose purpose is to find where something is (a file, a listing, a search): at most 4,000 tokens of output per call. Take a table of contents or an `rg` hit list first, then the lines it names. **Targeted read**, one whose target is known (a line range, a ticket, a diff hunk): as long as the target, never the whole file around it.
- **Reference documents** (`CONTEXT.md`, `.ai/PROJECT_ARCHITECTURE.md`, `.ai/DESIGN_KIT.md`, ADRs, other skills' `SKILL.md`) are queried with `rg` for the term in hand, and only when a decision of the main session needs them: the implementer and the reviewers read the project on their own. Material not on disk (a Linear issue with its comments, a tool listing, a search result) is written to a scratch file first and queried the same way.
- **Linear reads** carry a field list and a limit sized to the answer. The forms allowed in Preflight are written there; the scratch spec in [Review](#4-review-one-sub-issue) is the one full read of a Sub-issue. A search, for instance before a follow-up issue, is `list_issues` with `query`, `team`, `project`, `fields: ["id", "title", "status"]`, `limit: 10`.
- **Subagents** start with no inherited context: the brief is everything they know. On Codex pass `fork_turns: "none"` to `spawn_agent`; its default, `all`, copies this whole conversation into the child. The roster is what this session spawned: task names and outcomes come back in the spawn and wait results, so `list_agents` never runs. Wait with the longest timeout the harness accepts; a timed-out wait is followed by another wait.
- **Turn budget**: 120 turns per run, a turn being one model response, tool calls counted as the proxy. Check the count before each implementer dispatch and before each fix round. Over budget, the Linear state is already resumable: write the [report](#report), then ask the user one question: continue in this session, with the context as large as it is, or stop here and rerun the skill in a fresh session, which resumes from the first unfinished Sub-issue. Recommend the fresh session.

## 1. Preflight

Nothing touches git until every check below passes. Any failed check ends the run with a message naming what failed.

First, one question to the user: which model the implementer runs on, and which the reviewers run on. Offer the session's own model as the default for both. Keep both answers for the whole run: every implementer spawn passes the implementer model, every reviewer spawn the reviewer model (Codex: `model` on `spawn_agent`, accepted only with `fork_turns: "none"`; Claude Code: `model` on the Agent call, ignored by a fork). A name the harness rejects stops the run at that spawn with the harness error.

Then the checks:

1. Resolve the argument (identifier or URL) to the Parent issue with `get_issue`. Keep its identifier, team, and `gitBranchName`.
2. List its Sub-issues: `list_issues` with `parentId` = the parent, `fields: ["id", "title", "status", "statusType", "labels", "createdAt"]`, `orderBy: "createdAt"`, default limit; follow `cursor` when a page is full. No Sub-issues: stop.
3. Nested check, one call per Sub-issue: `list_issues` with `parentId` = that Sub-issue, `fields: ["id"]`, `limit: 1`. Any result means nested Sub-issues: stop.
4. Resolve the `ready-for-agent` label string: the mapping in `docs/agents/triage-labels.md` when the repo has one, else the literal `ready-for-agent`.
5. Classify every Sub-issue:
   - `statusType` `completed` or `canceled`: **skipped**
   - anything else carrying the label: **queued**
   - anything else: **not ready**
   One or more not ready: stop and list them with state and labels.
6. Order the queue. One `get_issue` with `includeRelations: true` per queued Sub-issue; keep only its identifier, state, and "blocked by" list, and let the rest of the payload go: description and comments are read once, when the Sub-issue's turn comes (steps 3.2 and 4.1). Then sort topologically on the "blocked by" edges, blockers first, creation order as tie-break. A blocker that is skipped counts as satisfied. A cycle: stop and name the Sub-issues in it.
7. `list_issue_statuses` for the team. `In Progress`, `In Review`, and `Done` must all exist by name. Any missing: stop.
8. `git status --porcelain` must be empty. Otherwise ask the user whether to stash, commit, or abort. Never discard their changes.

Done when both models are chosen, the queue is ordered, every Sub-issue is classified, and the three states are resolved to ids.

## 2. Branch

The branch name is the Parent issue's `gitBranchName`.

- It exists (locally or on the remote): check it out. This is a **resume**. Commits already on it remain part of their Sub-issue's review range.
- It does not: update the default branch (`git pull`), then create the branch from it.

Keep the resolved default branch name. Per-Sub-issue review anchors are derived from it in the queue.

## 3. Work the queue

One Sub-issue at a time, in order. Dispatch nothing in parallel. Between Sub-issues, ask the user nothing.

1. Find `EXISTING`, every commit after the default branch whose subject contains `<ID>` as a standalone token, oldest first. Match non-alphanumeric boundaries, so both `(ASK-362)` and `ASK-362` match while `ASK-3620` does not. If any exist, `START` is the parent of the oldest one. Otherwise `START` is the current `HEAD`. Every review of this Sub-issue uses this same `START`.
2. `list_comments` with `issueId` = the Sub-issue, `orderBy: "createdAt"`; the latest comment carrying `START` is the checkpoint. Restore its `START`, `REVIEWED_HEAD`, open ordinary findings, pending delivery-boundary findings, local evidence paths, action, and last user decision. A checkpoint `START` must equal the derived `START`; otherwise stop and report the mismatch.
3. When the current state is `In Review` and `REVIEWED_HEAD` equals the current `HEAD`, dispatch no implementer and resume at its unresolved [ordinary escalation](#5-escalation) or [delivery-boundary escalation](#delivery-boundary-escalation). When there is no matching checkpoint or `HEAD` changed, go directly to [review](#4-review-one-sub-issue) from `START`.
4. Otherwise apply the [turn budget](#context-budget), then `save_issue`: state `In Progress`. Record `DISPATCH_HEAD`, then spawn the implementer with no inherited context, on the implementer model, with [IMPLEMENTER-BRIEF.md](IMPLEMENTER-BRIEF.md) as its whole prompt, including `EXISTING` and `START`. Wait for it.
5. Act on its outcome:
   - **BLOCKED**: put the question to the user. Post question and answer as a comment on the Sub-issue. Send the answer to the same subagent (it keeps its context) and wait again.
   - **FAILED**: go to [Stopping](#stopping).
   - **DONE** with no commits after `DISPATCH_HEAD` and no `EXISTING`: treat as FAILED.
   - **DONE** with no commits after `DISPATCH_HEAD` and non-empty `EXISTING`: continue only when the implementer explicitly reports that the existing implementation already satisfies the ticket and lists successful verification commands. Otherwise treat it as FAILED.
   - **DONE**: continue.
6. `save_issue`: state `In Review`. Run the [review](#4-review-one-sub-issue).
7. `save_issue`: state `Done`. Comment in Italian: branch, each commit as SHA and subject, one line on the review outcome, and any delivery-boundary decision with its local evidence paths. Labels stay as they are.

## 4. Review one Sub-issue

1. `get_issue` and `list_comments` (`orderBy: "createdAt"`) on the Sub-issue, once per review; write title, description, and comments to a scratch file: that file is the spec, and the reviewers read it from disk.
2. Call the Skill tool with "code-review": fixed point `START`, the scratch file as the spec. Its two reviewers start with no inherited context, on the reviewer model. Record the current `HEAD` as `REVIEWED_HEAD`.
3. Classify findings. **Blocking**: every Spec-axis finding (missing, partial, or wrong requirement) and every breach of a documented repo standard. **Non-blocking**: smells and judgement calls. A blocking Spec finding is a **delivery-boundary finding** when its only completion requires pushing, opening or updating a pull request, or attaching an artifact to a remote pull request. This skill never performs those operations. Store every other finding in `OPEN_ORDINARY`.
4. Add every delivery-boundary finding to `PENDING_DELIVERY`; never replace this set with a later review's output. Only an explicit Accept or Defer decision clears an entry. Every other finding is ordinary.
5. Before a fix round, apply the [turn budget](#context-budget), then `save_comment` an Italian checkpoint containing `START`, `REVIEWED_HEAD`, current `HEAD`, `OPEN_ORDINARY`, `PENDING_DELIVERY`, local evidence paths, and action `Fix`. Send only `OPEN_ORDINARY`, blocking ones marked, to the current implementer. On a review-only resume with no current implementer, spawn one fresh implementer with the dispatch brief first. Never include `PENDING_DELIVERY` in a fix prompt. When there are no ordinary findings, skip the fix round. A **FAILED** fix goes to [Stopping](#stopping).
6. The round had ordinary blocking findings: review again (steps 1 to 4). Two reviews per review cycle is the cap.
7. Ordinary blocking findings remain after the second review: use [ordinary escalation](#5-escalation). Accept, Guide, and Defer operate only on ordinary findings and never clear `PENDING_DELIVERY`.
8. When no ordinary blocking findings remain and `PENDING_DELIVERY` is non-empty, use [delivery-boundary escalation](#delivery-boundary-escalation).
9. Otherwise the review passed.

### Delivery-boundary escalation

Show `PENDING_DELIVERY` and every local evidence path returned by the implementer or restored from the checkpoint. Ask the user to choose:

- **Accept**: clear `PENDING_DELIVERY`; the review may complete and the caller moves the Sub-issue to Done as it is.
- **Defer**: create a follow-up issue in the same team and project for the pull-request-owning step, `relatedTo` the Sub-issue, with the local evidence paths; clear `PENDING_DELIVERY`; the review may complete and the run continues.
- **Stop**: the Sub-issue stays `In Review`; go to [Stopping](#stopping).

Before applying the choice, `save_comment` in Italian with `START`, current `HEAD`, every finding, the choice, and every local evidence path. When the user does not choose, record and apply Stop. `Guide` is not offered because an implementer working only on the local branch cannot complete a remote pull request operation.

## 5. Escalation

Show the user the ordinary blocking findings still open and ask which of these they want:

- **Accept**: treat the ordinary findings as accepted; continue to delivery-boundary escalation when `PENDING_DELIVERY` is non-empty, otherwise the review may complete.
- **Guide**: the user gives direction. If no current implementer exists, spawn one fresh implementer with `EXISTING`, `START`, and the dispatch brief. Send only the ordinary findings for one more fix round, then run up to two more reviews. Continue to delivery-boundary escalation when the ordinary findings clear and `PENDING_DELIVERY` is non-empty.
- **Defer**: create a follow-up issue in the same team and project for the ordinary findings, `relatedTo` the Sub-issue; continue to delivery-boundary escalation when `PENDING_DELIVERY` is non-empty, otherwise the review may complete.
- **Stop**: the Sub-issue stays `In Review`; go to [Stopping](#stopping).

Before applying the choice, `save_comment` in Italian with `START`, current `HEAD`, the ordinary findings, the choice, and any still-pending delivery-boundary findings and evidence paths. When the user does not choose, record and apply Stop. A guided fix prompt contains only ordinary findings.

## Stopping

The run stops at the first FAILED or at a Stop in escalation. The Sub-issue keeps the state it had, the branch and its commits stay exactly as they are, nothing is reset. Escalation Stops already have a Linear checkpoint; on FAILED, add an Italian checkpoint with `START`, `REVIEWED_HEAD` when a review ran, current `HEAD`, the failure, `OPEN_ORDINARY`, `PENDING_DELIVERY`, action `Failed`, and every local evidence path before writing the [report](#report). On rerun, equality is tested against `REVIEWED_HEAD`: unchanged reviewed code resumes escalation, while a newer current `HEAD` is reviewed again from `START`.

Rerunning the skill on the same Parent issue resumes: Done Sub-issues are skipped, the branch is reused, `START` still precedes the oldest Sub-issue commit, and an unchanged `In Review` checkpoint resumes at escalation without manufacturing another commit.

## Report

At the end, complete or stopped, tell the user:

- the branch
- per Sub-issue: done, skipped, failed, or in review; its commits; one line on its review
- every question answered during the run
- every delivery-boundary finding, the user's decision, and all local evidence paths
- the remaining queue, when stopped
- the turn count, when the turn budget stopped the run
- the models the implementer and the reviewers ran on

The skill never pushes, never opens a PR, and never modifies the Parent issue. Suggest `/code-review main` on the whole branch before the PR.

## Why the main session runs the review

`implement` has the implementer call `code-review`, which spawns two fresh reviewer subagents. A subagent cannot spawn subagents, so an implementer subagent calling `code-review` would review its own work inside its own context. Running the review from the main session keeps the reviewers fresh. The fix still goes back to the implementer because it already holds the context to make it cheaply.
