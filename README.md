NoIfApology does not decide who was right, and it does not judge whether an apology is warm enough. It asks one thing: does the answer hold the speaker to account for what they did — and only such an answer closes the complaint.

# NoIfApology

GenLayer Intelligent Contract · StudioNet (chain 61999) · py-genlayer v0.2 (`# v0.2.16`)

**The contract holds no money and moves no money.** It records grievances, the answers given to them and how each one
ended. It enforces nothing off chain.

| | |
|---|---|
| Contract | `contracts/NoIfApology.py` |
| Address (StudioNet) | `0x8865b318888Bd1BdFc84e25525EBeD2d4308415E` ([explorer](https://explorer-studio.genlayer.com/address/0x8865b318888Bd1BdFc84e25525EBeD2d4308415E)) |
| Deploy tx | [`0x3a5ba8d3ec23132a84b40e3304950738918334290173d1cf6fb35ac845f611e6`](https://explorer-studio.genlayer.com/tx/0x3a5ba8d3ec23132a84b40e3304950738918334290173d1cf6fb35ac845f611e6) |
| Source SHA-256 | `4184c9e839a4c647bf8c439f3522e84a76db8c3efe3587718f8d924de321503f` |
| Tested on-chain | `TESTING.md` — 15 transactions, one hash each; every must-verify row passed |

## What it does

A complainant files a short grievance against one wallet. Only that wallet may answer it, at most three times.
Validators read each answer **once**, together with the complaint, and decide one thing:

- `OWNS_IT` — the answer holds the speaker to account for the conduct complained of, as the speaker's own conduct;
- `DEFLECTS` — it does not.

| | `OWNS_IT` | `DEFLECTS` |
|---|---|---|
| Grievance | **RESOLVED**, for good | stays **OPEN**, one attempt spent |
| Third answer that deflects | — | **CLOSED_UNANSWERED**, for good |
| Respondent's public standing | `resolved_by_owning + 1` | on closing: `closed_unanswered + 1` |

The same answer text (after whitespace is collapsed) cannot be used twice on one grievance, so an attempt cannot be
replayed. A complainant can hold only one open grievance against the same wallet at a time, and can withdraw it while it
is open; a withdrawn grievance leaves the standing untouched.

`I'm sorry that my comment about your pull request was rude.` and `I'm sorry that you felt my comment about your pull
request was rude.` differ by two words. The first owns the remark; the second turns it into the listener's feeling. That
pair is the core test of the contract.

## Methods

| Write | Who | Checks, in order |
|---|---|---|
| `file_grievance(respondent_wallet, complaint)` | anyone | wallet → not yourself → complaint 1–120 characters → no reserved token → no open grievance against that wallet → not filed before |
| `answer(grievance_id, text)` | the named respondent | id → OPEN → respondent → text 1–200 characters → no reserved token → not given before → **the only model call** |
| `withdraw_grievance(grievance_id)` | the complainant | id → OPEN → complainant |

Views (JSON strings; an unknown id returns `"{}"`, never a revert): `get_grievance` (every field plus every answer with
its outcome, and `attempts_left`), `get_standing(wallet)`, `get_rubric`, `get_limits`. There is no preview or dry-run view.

Grievance id: `keccak256("NO_IF_APOLOGY:GRIEVANCE:V1|" + complainant_lower + "|" + respondent_lower + "|" + len(complaint) + "|" + complaint)`,
where `complaint` has all whitespace collapsed (Python `" ".join(text.split())`).

## Design choices

- **Fail-safe is `DEFLECTS` — the grievance stays open.** A wrong `OWNS_IT` closes the complainant's grievance on an
  apology that was not one, and nothing reopens it. A wrong `DEFLECTS` costs the respondent one attempt to say it more
  plainly. When the model's output is unusable, the contract keeps the grievance open.
- **The model sees the complaint and the answer, nothing else.** No wallet, no attempt count, no state, nothing about what
  the contract will do next. `OWNS_IT`, `DEFLECTS` and the four tag markers are refused in any letter case before the
  model is called, and stripped to a fixed point inside the prompt.
- **A cap on attempts, not on time.** Three answers that do not own the conduct close the grievance as unanswered; the
  respondent's standing shows both endings, so silence and deflection are visible next to answers that owned the conduct.
- **No re-grinding.** An answer cannot be resent with different spacing for another reading, and the complaint text is
  part of the grievance id, so the same complaint cannot be refiled after it closed.
- **The rubric names the task, not the answers.** It never lists the ways an apology can push the fault elsewhere; those
  are exactly what the test pairs check.

## How to try it in GenLayer Studio

Use **two wallets of your own**: complainant (A) and respondent (B).

1. A: `file_grievance(<B>, "Called my pull request lazy in the public channel")`.
2. B: `answer(<id>, "I'm sorry that you felt my comment about your pull request was rude.")` → `DEFLECTS`, still OPEN.
3. B: `answer(<id>, "I'm sorry that my comment about your pull request was rude.")` → `OWNS_IT`, RESOLVED.
4. Read `get_grievance(<id>)` and `get_standing(<B>)`.

The grievance id is computed from the formula above (any keccak256 tool works). Text may not contain `OWNS_IT` or
`DEFLECTS` in any letter case. Keep answers under about 160 characters so the calldata stays under 255 bytes.

## Honest limitation

1. **The contract holds no money and enforces nothing off chain.** A resolved grievance is a record, not a remedy.
2. **It does not judge whether the complaint was fair.** Anyone can file a grievance about any wallet; the respondent may
   simply not answer.
3. **It reads words, not hearts.** A ready-made apology that owns the conduct is `OWNS_IT`; sincerity is not measured.
4. **A wrong `OWNS_IT` is the main risk** — it closes a grievance on an apology that was not one, and nothing reopens it.
   Nets: the fail-safe, and a grievance closes only on a clear answer from the model.
5. **There is no deadline.** A respondent who never answers leaves the grievance open for good.
6. **Answers longer than about 161 characters are not proven.** The contract accepts up to 200 characters, but at that
   length `answer` measures 296 bytes of calldata, over the 255-byte RPC limit.

License: MIT.
