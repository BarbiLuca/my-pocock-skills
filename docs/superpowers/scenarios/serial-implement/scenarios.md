# Serial Implement pressure scenarios

These scenarios exercise the user-invoked `serial-implement` skill while it is
in the `in-progress` bucket. Reports record observable decisions rather than
matching exact prose.

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
