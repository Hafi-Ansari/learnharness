# Fork notes: `fix/empty-answer-continuation`

This branch exists only in this fork. It carries one bug fix that upstream has not merged yet. Read this before updating, rebasing or deleting the branch.

Written 2026-11-13, on top of upstream `main` at v0.6.1 (`cb25c8e`).

## The bug

In a teaching session, the learner submits an answer to a question and nothing happens. The lesson only continues after the learner sends another message such as "ok" or "continue", sometimes several times.

What actually happens:

1. The answer is saved correctly as the question tool's result.
2. The model is called and returns a **completed but empty** reply: one empty text block marked `final_answer`, `stopReason: "stop"`, about 4 output tokens, no tool call, no error.
3. LearnHarness saves that as a normal finished turn. Nothing detects it, nothing retries, and the UI has nothing to show.

Observed in this fork's own sessions on the ChatGPT subscription connection with `gpt-6-astra` as the teacher: about a third of submitted answers got an empty reply. Upstream users report the same with `gpt-6.1-sol`. It is model behaviour, so it is not tied to `npm run dev` versus `npm start`, or to one operating system.

Upstream tracking:

- Issues: [#482](https://github.com/amosblomqvist/learnharness/issues/482) (the exact bug), [#455](https://github.com/amosblomqvist/learnharness/issues/455), [#628](https://github.com/amosblomqvist/learnharness/issues/628), [#618](https://github.com/amosblomqvist/learnharness/issues/618)
- Fix this branch ports: [PR #480](https://github.com/amosblomqvist/learnharness/pull/480) by kocendavid, open and unreviewed as of 2026-11-13
- A different, complementary approach, not included here: [PR #678](https://github.com/amosblomqvist/learnharness/pull/678)

## What the branch changes

Two commits cherry-picked from [BSathvik/learnharness#1](https://github.com/BSathvik/learnharness/pull/1), which is PR #480 rebased onto v0.5.0 plus one extra regression test. Original authorship is kept. A third commit adds this file.

Behaviour after the fix:

- When the reply to a question answer is empty, the same agent makes **one** automatic follow-up request, with a system reminder to respond to the saved answer.
- If that reply is also empty, the turn becomes a saved error with the existing **Retry** button. Retry keeps the learner's answer.
- Provider errors, aborts and practice sessions behave as before.

| File | Change |
|---|---|
| `src/agent/teacher-agent.ts` | The recovery logic: `finishTurn` requests one continuation, the next-turn hook adds the reminder, and a subscriber turns a second empty reply into an error |
| `src/core/native-messages.ts` | New `isEmptyAssistantCompletion` helper |
| `src/core/input-events.ts` | Retry also skips the empty attempts when it rebuilds context |
| `src/agent/subscription-request-cycle.ts` | New `oweForegroundContinuation()` so the follow-up is admitted as a foreground request |
| `src/agent/prompts/shared/20-question-tools.md` | The "ONE question per turn" line now says to react to the answer and not end with an empty response |
| `docs/architecture.md` | One paragraph describing the recovery |
| `tests/chat-flow.test.ts`, `tests/chatgpt-request-cycle.test.ts`, `tests/retry-failed-turn.test.ts` | Regression tests for all of the above |

### The one non-mechanical merge decision

v0.6 already defined `prepareNextTurnWithContext` in `teacher-agent.ts`, to deliver side-question updates (`opts.contextUpdates`). PR #480 defines the same hook for its reminder. An object can only have one, so this port **combines them into a single hook**: side-question updates first, then the reminder when it applies. If a future upstream change touches that hook, keep both behaviours.

The other conflicts were "keep both sides" merges.

## What it does not fix

- The model replying with a short acknowledgement and then stopping without the next question (upstream #455).
- Empty replies to a plain learner message such as "continue". Recovery only runs when the turn starts from a question answer.
- Old chats keep their saved empty replies in history, and those are still replayed to the model. Test the fix in a fresh chat.
- Each recovered stall still costs one wasted model call plus the follow-up.

## Verification status

Run on Windows, Node 26.12.1, 2026-11-13:

- `npm run typecheck`: clean.
- ESLint on the changed files: clean.
- The three touched suites: 104 of 104 pass.
- Full suite: 67 failures on the branch against 66 on unmodified `main`. The failing tests are the same environment-related ones on both (symlinks, file modes, an installed Claude CLI). The one extra was a demo-helper timing test that passes when rerun alone.

**Not yet verified:** a live teaching session with a real provider. Until someone confirms it there, treat the fix as tested but unproven.

To rerun the focused tests:

```sh
NODE_ENV=test npx tsx --test tests/chatgpt-request-cycle.test.ts tests/retry-failed-turn.test.ts tests/chat-flow.test.ts
```

Before running the full suite, read the "Checks" section of `docs/development.md`. Set `LH_DATA_DIR` to a throwaway folder for the run so no test can touch real learning data.

## Using the branch on another device

```sh
git fetch origin
git switch fix/empty-answer-continuation
```

No dependency changed, so `npm install` and `npm run plugins:setup` are only needed if the device was not already set up for v0.6.1. The branch does not change any saved-data format, so it is safe to move learning data between a device on this branch and one on plain v0.6.1.

## Keeping it up to date

When upstream releases a new version:

1. Update `main` as usual.
2. `git switch fix/empty-answer-continuation` then `git rebase main`.
3. Resolve conflicts if any. `src/agent/teacher-agent.ts` is the likely place; see the merge decision above.
4. Rerun the focused tests and the typecheck.
5. The rebase rewrites the branch, so pushing it needs `git push --force-with-lease`. Ask the owner before doing that, and update the other device afterwards with `git fetch` and `git reset --hard origin/fix/empty-answer-continuation` on this branch only.

## When to delete it

Drop the branch and go back to `main` once upstream merges PR #480 or fixes the empty-reply stall another way. Check that `isEmptyAssistantCompletion`, or an equivalent check, exists on upstream `main` before deleting.
