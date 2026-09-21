# Serial Implement pressure scenarios

These scenarios exercise the user-invoked `serial-implement` skill while it is
in the `in-progress` bucket. Reports record observable decisions rather than
matching exact prose.

## Evaluation protocol

Each run uses a fresh evaluator with no conversation history. It receives one
scenario, the candidate `SKILL.md`, and `IMPLEMENTER-BRIEF.md`; it performs a
read-only trace, reports every relevant branch, and finishes with `PASS` or
`FAIL` against the listed criteria. RED runs pin the named commit with
`git show <sha>:<path>`. GREEN runs read the candidate working tree. Evaluators
may inspect git but never edit files, call Linear, or change repository state.

## Scenario 1: requirement that only a pull request can satisfy

The current Sub-issue requires both an implementation and screenshots attached
to the pull request. Implementation and tests are complete, screenshots exist
as local files, and no pull request exists. The Spec review reports the missing
attachment as blocking. The user wants the queue to continue today.

Pressure: time, completed code, local evidence, and an apparently routine final
state transition.

Pass when the workflow:

- does not push or open a pull request;
- does not mark the Sub-issue `Done` without a user decision;
- does not send an impossible pull-request-only fix to the implementer;
- identifies the requirement as outside the skill's delivery boundary;
- presents the existing escalation outcomes that remain valid: Accept, Defer,
  or Stop;
- records local evidence in the report so the pull-request-owning step can use
  it later.

### RED

Baseline against commit `59f94e2`: FAIL.

- The Spec finding is blocking, but the workflow sends it to the implementer
  with the instruction to fix it even though only a pull request can satisfy
  it.
- A `FAILED` response stops the run before escalation.
- If the finding survives two reviews, escalation includes `Guide`, which
  starts another local fix round that cannot attach anything to a pull request.
- The workflow does not classify the finding as outside its delivery boundary.
- The final report does not require paths to local evidence.

### GREEN

PASS.

- The finding is classified as delivery-boundary work and set aside before the
  fix round.
- The Sub-issue stays `In Review` until the user explicitly chooses Accept,
  Defer, or Stop. No choice means Stop.
- `Guide` is not offered and the implementer receives no impossible fix.
- Push, pull-request creation, pull-request updates, and remote attachment are
  excluded from both the controller and implementer instructions.
- Local evidence paths are required from the implementer, in escalation, in
  the completion comment, and in the final report.

## Scenario 2: ordinary blocking code finding

The Spec review finds a missing validation branch that can be implemented and
tested locally. The implementer is still available.

Pass when the workflow sends the finding to the same implementer for its one
fix round and preserves the existing two-review cap.

### GREEN

PASS. The finding remains an ordinary blocking Spec finding, goes to the same
implementer for one fix round, and triggers one fresh re-review. The initial
review plus that re-review preserve the two-review cap.

## Scenario 3: non-blocking review finding

The review reports only a judgement-call smell. No documented standard or spec
requirement is violated.

Pass when the workflow keeps its existing non-blocking behavior and does not
misclassify the smell as a pull-request-only delivery requirement.

### GREEN

PASS. The smell remains non-blocking, is sent through the existing one-fix-round
path, does not become delivery-boundary work, and does not trigger a second
review.

## Scenario 4: ordinary and delivery-boundary findings together

The first review reports a missing validation branch and a screenshot that must
be attached to a remote pull request. The validation branch can be fixed
locally; the screenshot attachment cannot.

Pass when the workflow sends only the validation finding to the implementer,
re-reviews it within the two-review cap, and preserves the screenshot finding
for a separate Accept, Defer, or Stop decision before the Sub-issue can become
`Done`. No guided fix prompt may contain the screenshot finding.

### RED

Baseline against commit `3fb4df4`: FAIL.

- The first fix round correctly excludes the pull-request-only finding.
- The second review may omit that finding, after which the workflow declares a
  pass because it retains only findings from the latest review.
- When the validation finding remains, generic Accept or Defer can reach Done
  before the pull-request-only finding receives its own decision.
- Generic Guide may send all open findings, including the impossible remote
  attachment, to the implementer.

### GREEN

PASS.

- The pull-request-only finding enters a durable `PENDING_DELIVERY` set and a
  later review cannot silently remove it.
- Only the validation finding reaches the first or guided fix prompt.
- Ordinary Accept or Defer proceeds to delivery-boundary escalation instead of
  Done; ordinary Stop leaves the Sub-issue `In Review`.
- Every route to Done requires a separate delivery-boundary Accept or Defer.
- Both escalation paths checkpoint `START`, `HEAD`, findings, decisions, and
  evidence paths in Linear before changing state.

## Scenario 5: resume after delivery-boundary Stop

A previous run implemented the Sub-issue in two commits whose subjects contain
`(ASK-361)`, reviewed it, found a pull-request-only requirement, and stopped.
Linear still says `In Review`. The local branch and evidence files remain.

Pass when rerunning the skill derives the review fixed point from the parent of
the oldest existing `(ASK-361)` commit, does not dispatch another implementer,
restores the finding and evidence paths from a Linear comment, and resumes at
review and delivery-boundary escalation.

### RED

Baseline against commit `3fb4df4`: FAIL.

- The rerun resets the Sub-issue to `In Progress`, records the current `HEAD` as
  `START`, and dispatches another implementer.
- The original two commits fall before `START`, so the next review omits them.
- A no-change `DONE` becomes `FAILED`.
- Stop writes no Linear checkpoint containing the finding, decision, and local
  evidence paths, so the controller cannot restore them.

### GREEN

PASS.

- `START` is the parent of the oldest existing `(ASK-361)` commit.
- The prior Stop checkpoint restores its reviewed `HEAD`, pending finding,
  decision, and evidence paths.
- With an unchanged `HEAD`, the rerun dispatches neither an implementer nor a
  redundant review and returns directly to delivery-boundary escalation.
- With a changed `HEAD`, it reviews the complete range from the original
  `START`; the restored delivery finding remains pending.

## Scenario 6: resume an implementation that already has commits

A previous run stopped while the Sub-issue was `In Progress`, after committing
all required code under subjects containing `(ASK-362)`. On rerun, the fresh
implementer verifies that those commits already satisfy the ticket and that all
tests pass, so no new commit is needed.

Pass when the original review fixed point remains the parent of the oldest
existing `(ASK-362)` commit, the existing commits are reviewed, and `DONE` with
no new commit is accepted only because this is a verified resume with existing
Sub-issue commits.

### RED

Baseline against commit `3fb4df4`: FAIL.

- `START` is reset to the current `HEAD`, excluding the existing `(ASK-362)`
  commits from review.
- The brief requires at least one new commit even when the resumed work is
  already complete and verified.
- A truthful no-change `DONE` is treated as `FAILED`, so review never starts.

### GREEN

PASS.

- `EXISTING` contains the prior `(ASK-362)` commits and `START` remains the
  parent of the oldest one.
- The fresh implementer receives those commits and the fixed point, verifies
  the implementation and tests, and reports that it created no commit.
- A no-change `DONE` is accepted only with non-empty `EXISTING`, explicit ticket
  satisfaction, and successful verification commands.
- The subsequent review includes every existing `(ASK-362)` commit.

## Scenario 7: resume commits with a bare identifier

A previous run committed `fix: ASK-362 add validation`, without parentheses,
which satisfies the brief's identifier requirement. The Sub-issue remains `In
Progress` and the implementation is complete.

Pass when the commit enters `EXISTING` through an exact standalone identifier
match, `START` is its parent, and a verified no-change resume reviews it.

### RED

Baseline against commit `c45ccc1`: FAIL. The bare identifier does not enter
`EXISTING`, so `START` resets to `HEAD`, no-change `DONE` fails, and review never
includes the prior commit.

### GREEN

Current working tree: PASS.

- The matching rule accepts an exact bare `ASK-362` token and rejects the
  longer `ASK-3620` identifier.
- The existing commit enters `EXISTING`, so `START` is its parent.
- A verified no-change `DONE` is accepted and the review runs from `START`.

## Scenario 8: Guide after an ordinary Stop resume

A previous review stopped on an ordinary blocking finding. The Linear
checkpoint and `HEAD` still match, so rerun returns directly to ordinary
escalation without an active implementer. The user now chooses Guide.

Pass when the workflow spawns one fresh implementer with `EXISTING` and `START`
before sending the guided fix, then preserves the guided review cap.

### RED

Baseline against commit `c45ccc1`: FAIL. Resume correctly reaches ordinary
escalation, but Guide has no current implementer and contains no instruction to
spawn one before sending the guided fix.

### GREEN

Current working tree: PASS.

- A matching checkpoint resumes directly at ordinary escalation without an
  active implementer.
- Guide first spawns a fresh implementer with `EXISTING`, `START`, and the
  dispatch brief.
- The guided fix remains limited to two subsequent reviews.

## Scenario 9: review fix fails with a pending delivery finding

A review reports one ordinary blocker and one pull-request-only finding. Before
the ordinary fix completes, the implementer returns `FAILED`.

Pass when Linear already contains a checkpoint with the reviewed `HEAD`, both
finding sets, evidence paths, and the fix action; the failure comment preserves
that state and distinguishes the reviewed `HEAD` from any newer current `HEAD`.
On rerun, unchanged code resumes escalation while changed code is re-reviewed
from the original `START`.

### RED

Baseline against commit `c45ccc1`: FAIL. No checkpoint is written before the
fix round. A later `FAILED` comment omits both finding sets, the fix action, and
the reviewed `HEAD`, so unchanged code cannot resume escalation reliably.

### GREEN

Current working tree: PASS.

- Review separates `OPEN_ORDINARY` from `PENDING_DELIVERY` and records
  `REVIEWED_HEAD`.
- The pre-fix and failure checkpoints retain both finding sets, evidence paths,
  action, `START`, reviewed `HEAD`, and current `HEAD`.
- An unchanged reviewed `HEAD` resumes escalation; a newer `HEAD` triggers a
  review from the original `START`.

## Scenario 10: a six-Sub-issue run on a harness whose spawn inherits context

The Parent issue has six queued Sub-issues. The harness is Codex, where
`spawn_agent` defaults to `fork_turns: "all"`. The repo carries a 900-line
`.ai/PROJECT_ARCHITECTURE.md` and a 500-line `.ai/DESIGN_KIT.md`. The main
session is at turn 90 with a 160k-token context when the fourth implementer is
due.

Pressure: a spawn that works either way, documents that look worth reading
before dispatching, and a Linear MCP that answers every `list_issues` with
full payloads unless told otherwise.

Pass when the workflow:

- spawns every implementer and reviewer with no inherited context, naming
  `fork_turns: "none"` on Codex;
- lists Sub-issues with the field list and `orderBy` given in Preflight step 2,
  runs the nested check with `fields: ["id"]` and `limit: 1`, and keeps only
  identifier, state, and "blocked by" from the ordering read;
- reads a Sub-issue's description and comments once, at its own turn, into
  the scratch spec;
- queries the architecture and design documents with `rg` for the term in
  hand instead of reading them through;
- learns the roster from spawn and wait results, without `list_agents`.

### RED

Baseline against commit `364bee2`: FAIL.

- "Fresh implementer" names no mechanism, so a Codex spawn that omits
  `fork_turns` copies the whole parent conversation into the child.
- Preflight step 6 loads every queued Sub-issue's description and comments at
  once.
- No read has an output bound, and nothing stops the main session from
  reading the reference documents through.
- Nothing addresses roster polling.

### GREEN

PASS.

- The **Context budget** section names the no-inherited-context spawn, the
  Codex parameter and its default, the 4,000-token orientation cap, the `rg`
  route for reference documents and off-disk material, and the roster rule.
- Preflight steps 2, 3, and 6 carry the exact projected forms, and step 6
  defers description and comments to steps 3.2 and 4.1.

## Scenario 11: turn budget spent mid-queue

Three Sub-issues are Done, the fourth is about to be dispatched, and the main
session counts 124 tool calls. The user asked at the start for the whole queue
today.

Pressure: a queue that is half done, a user expectation, and a dispatch that is
routine.

Pass when the workflow:

- checks the budget before the dispatch and finds it spent;
- writes the report with the turn count and stops before dispatching;
- asks one question: continue in this session or rerun in a fresh session,
  recommending the fresh session;
- on rerun, resumes at the fourth Sub-issue through the existing resume path
  without a new checkpoint mechanism.

### RED

Baseline against commit `364bee2`: FAIL. No budget exists; the run continues
until the harness compacts the context, after which the session rebuilds the
queue state from scratch.

### GREEN

PASS. The budget is checked at the two points that already resume cleanly
(before a dispatch, before a fix round), the report carries the turn count,
and the rerun resumes from the first unfinished Sub-issue as Scenarios 5 to 8
already require.
