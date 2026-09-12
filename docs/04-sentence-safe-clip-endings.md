# Architecture — Sentence-safe clip ending for the podcast reel

**Workflow:** `YOUR_WORKFLOW_ID` — *Podcast Video Automation v1 - CONTROLLED RUNTIME TEST - INACTIVE*
**Document version:** 1
**Date:** 2026-09-12
**Author:** Designer
**Scope:** one node — `Code - Parse Validate Moment Copy And Build VTT Render Contract`
**Status:** proposed, not built. Designer has no n8n write tools and made no n8n change.

---

## 0. The requirement

Leo, verbatim:

> "the reel does not need to be strictly 45 seconds. The important thing is it does not cut
> the important scene or words midway when it reaches the end."

Plain reading: the length is now free inside the limits the workflow already enforces.
What matters is that the reel stops at a place a listener would recognise as an ending.

## 1. Why the current code cannot guarantee that

The only node that decides clip boundaries is
`Code - Parse Validate Moment Copy And Build VTT Render Contract`. Its `jsCode` was read from
`workflows/podcast-video-automation-v1/runtime-test.json` line 1196. Today it:

- takes `start_ms` / `end_ms` straight from the OpenAI moment JSON;
- throws `MOMENT_DURATION_OUT_OF_BOUNDS` unless the span is 30–60 s;
- throws `MOMENT_BOUNDARY_NOT_WORD_ALIGNED` unless the first and last AssemblyAI word inside
  the span sit within 100 ms of the proposed bounds;
- sets `d = (end_ms - start_ms)/1000`, and the Shotstack source clip uses
  `trim: start_ms/1000`, `length: d`;
- builds caption groups of four words with offsets relative to `start_ms`.

`MOMENT_BOUNDARY_NOT_WORD_ALIGNED` proves a **word** boundary. It never proves a **sentence**
boundary. In execution 525 the end happened to be a sentence end because OpenAI chose one, not
because anything checked. On a 47-minute podcast the model will land mid-sentence.

### Evidence already measured (not re-derived here)

From execution 525, the reel Leo accepted:

| Fact | Value |
|---|---|
| moment start | 55 754 ms |
| moment end | 100 739 ms |
| `d` | 44.985 s |
| words | 111 |
| caption groups | 28 (27 of four words, one of three) |
| last word | `here.`, spanning 100 610 – 100 739 ms |
| tail after the final syllable | **0 ms** |
| next transcript word | `It's` at 100 948 ms (209 ms later) |
| word before the moment | `these?` ending at 54 806 ms (948 ms of clear space) |
| audio RMS, last second | 5132 @ 44.70 s → 1555 @ 44.90 s → 568 @ 44.95 s |
| frame at 44.70 s | caption `not even here.`, highlight still on `even` |

The cut lands on the decay, not on a truncated syllable. That is why the reel was acceptable —
by luck, not by design.

### Downstream constraints confirmed by reading the workflow JSON

All four were re-checked directly, not taken on trust.

1. `Code - Validate Media Probe And Build TEST Evidence` (line 2390) asserts
   `duration_in_contract: Number.isFinite(duration) && duration >= 30 && duration <= 60`
   against the FFprobe duration of the **finished MP4**. A final duration above 60 s fails the
   run *after* the money is spent.
2. `Code - Refuse Duplicate Shotstack Submission And Build Claim` (line 1654) and
   `Code - Enforce Shotstack Claim And Cost Ceiling` (line 1868) both compute
   `projected = (assembly_estimate_usd + openai_reserved_usd + p.moment.duration_seconds/60*0.30).toFixed(4)`
   and refuse above `cost_ceiling_usd` (0.50). The second compares its own result with the
   first's to `< 0.000001`. **Both read the same field, `p.moment.duration_seconds`, from the
   same node.** So they stay identical automatically, whatever that field holds.
3. `Airtable - Upsert TEST Selected Moment` (lines 1227–1235) writes
   `moment.start_ms`, `moment.end_ms`, `moment.duration_seconds` (inside `scores_json`) and
   `moment.words`.
4. Caption group offsets are relative to the clip start. If the clip start moves, every offset
   must move with it. `GAP_MS = 40` and the `VTT_CUE_GAP_COLLAPSE` fallback are settled and
   must survive untouched.

Contract values used below, read from `Code - Define Approved Runtime Test Contract` (line 27):
`assembly_estimate_usd = 0.0112`, `openai_reserved_usd = 0.05`, `cost_ceiling_usd = 0.50`,
`source_duration_seconds = 175.999`.

### Node schema check

`get_node` on `nodes-base.code` confirms version 2, `mode: runOnceForAllItems`,
`language: javaScript`, `jsCode` — exactly what the node already has. No parameter drift. No
new node is introduced, so no other schema lookup applies.

`search_templates` returned nothing for this shape. That is expected: this is a line-level edit
to bespoke transcript arithmetic, not a workflow pattern. No template was used.

---

## 2. Decisions

### D1 — Snap the clip end forward to the next sentence-ending word. **Adopted.**

When the model's chosen last word does not end a sentence, scan forward through the full
AssemblyAI word array and move the end to the first word that does.

Reason: it is the only change that makes "does not cut words midway" a property of the code
rather than a property of luck. It moves the end forward only, so no chosen content is lost.

**The exact rule.** AssemblyAI returns punctuation inside `word.text` (`here.`, `glasses,`,
`these?`). A word ends a sentence when all of these hold:

1. After stripping any trailing closing quotes or brackets — `"` `'` `”` `’` `)` `]` `}` — the
   token's last character is one of `.` `!` `?` `…`.
2. The remaining core (terminator stripped too) is at least 2 characters. This rejects initials
   such as `J.` in `J. R. R. Tolkien`.
3. The core, lower-cased, is not in a small abbreviation list:
   `mr, mrs, ms, dr, prof, st, sr, jr, vs, etc, no, inc, ltd, co, u.s, e.g, i.e`.
4. **If the terminator is a full stop** (`.` only — `!` `?` `…` are never abbreviations): either
   there is no next word, or the next word, after stripping leading quotes and brackets, begins
   with an uppercase letter or a digit.

Condition 4 is the strongest filter and costs one line. `here.` followed by `It's` passes.
`U.S.` followed by `citizens` fails.

**Accepted residual risk:** an abbreviation outside the list and followed by a proper noun
(`Mr. Smith` if `mr` were removed from the list) would be read as a sentence end. The
consequence is a reel that stops slightly early at a plausible-sounding place. It can never
produce a mid-word cut or a bleed. That asymmetry is why the risk is acceptable.

**The existing `MOMENT_BOUNDARY_NOT_WORD_ALIGNED` check runs first, unchanged, against the
model's own proposal.** Its job is to prove the model's numbers correspond to real words. That
job is unaffected by anything below it.

### D2 — Search horizon, and what happens on give-up. **Root's proposal ratified.**

Horizon: `MAX_EXTEND_MS = 6000`.

Reason: conversational speech runs about 2.5 words per second, so 6 s is roughly 15 words —
enough to finish almost any sentence already in progress, and short enough that the reel never
drifts far from the moment the model actually judged to be good.

The search is bounded a second time by the duration budget (D6). A candidate is only acceptable
if `candidateWordEnd + TAIL_PAD_MS - clipStart <= budgetMs`. Using the *maximum* tail here is
deliberately conservative: the result can only ever come in under budget, never over. Being
conservative next to a cost ceiling is the correct bias.

**On give-up: fall back to the original word-aligned end. Do not drift. Do not throw.**

Three reasons:
- The model's end is already word-aligned and was chosen for meaning. Running 6 s into an
  unresolved new sentence is worse than the mild cut we have today.
- Throwing would destroy a run whose AssemblyAI and OpenAI work is already paid for, over a
  cosmetic defect. That trades real money for polish.
- Leo's requirement is "does not cut the important scene or words midway". The fallback still
  gets a whole-word boundary plus a tail pad, which is strictly better than today.

**The give-up must be visible, not silent.** Record the reason so QA and Tester can tell which
path ran. A silent fallback would let the feature quietly never fire on the real podcast and
nobody would know.

Recorded as `moment.ending.mode`, one of:

| Mode | Meaning |
|---|---|
| `already_sentence_end` | The model's own end was a sentence end. No search ran. This is what execution 525 produces. |
| `sentence_snap` | Search found a sentence end and the clip was extended. |
| `fallback_speaker_change` | Search hit the other host's line first (D3). |
| `fallback_no_sentence_end_within_horizon` | 6 s passed with no sentence end. |
| `fallback_duration_ceiling` | A sentence end may exist but it would breach the duration budget. |
| `fallback_transcript_end` | The transcript ran out. |

### D3 — The forward search stops at a speaker change. **Adopted.**

Reason: this is a two-host podcast. Running into the other host's line ends the reel on someone
else *starting* a thought, which is the same fault wearing a different hat. It also makes
`moment.speaker`, which is written to Airtable, untrue.

Rule: compare `String(word.speaker)` against `String(t.words[lastIdx].speaker)`. On any
difference, stop immediately, include none of that speaker's words, and fall back with mode
`fallback_speaker_change`. Using `String()` makes a `null` speaker compare consistently and
treats "diarised" versus "not diarised" as a change, which is the safe direction.

### D4 — Tail pad after the final word. **Root's clamp ratified, with a named guard constant.**

```
TAIL_PAD_MS        = 400
NEXT_WORD_GUARD_MS = 40
END_OF_MEDIA_GUARD_MS = 100

tail = clamp(0, TAIL_PAD_MS, nextWordStart - lastWordEnd - NEXT_WORD_GUARD_MS)
tail = min(tail, sourceDurationMs - lastWordEnd - END_OF_MEDIA_GUARD_MS)   // only if finite
```

`NEXT_WORD_GUARD_MS` is a **separate constant from `GAP_MS`** even though both are 40. They are
unrelated quantities — `GAP_MS` stops Shotstack merging touching VTT cues; this one keeps the
render clear of the next word's onset. Reusing the name would tie two independent things
together and invite one to be changed for the other's reasons.

Why 40 ms of guard: AssemblyAI's reported word start is the detected onset, and the acoustic
energy of a plosive or fricative begins slightly before that. 40 ms of clearance keeps the
first consonant of the next word out of the reel.

Why `TAIL_PAD_MS = 400`: 150–250 ms of silence is enough for a listener to register a full
stop, and broadcast practice leaves roughly 200–500 ms of room tone at the end of a cut. 400 ms
is the target; the clamp will often reduce it, and that is fine.

Why the source-duration clamp: `source_duration_seconds` is 175.999 s here. Without this, a
moment near the end of a recording would ask Shotstack to render past the end of the media.
Not binding in execution 525 (limit 75 160 ms) but it must be there for the real podcast.

**Is 169 ms enough in execution 525? Yes.** Justification, from the measured RMS: energy has
already fallen from 5132 to 568 — about 89% — by the point where the clip currently stops. The
pad does not need to carry a loud syllable to completion; it needs to cover the remaining decay
and then register as a pause. 169 ms does both. It also turns a cut that lands *on* the decay
into a cut that lands *after* it, which is the whole perceptual difference between "it stopped"
and "it ended".

**A small bleed is rejected outright.** A fragment of the next word — especially the other
host's first syllable — is far more noticeable and more damaging than a slightly short pause.
A clipped incoming syllable *is* a word cut midway. It is precisely the fault Leo asked to
remove, so trading one for the other is not a trade. **The tail may never exceed the gap to the
next word.** That is a hard rule, not a preference.

Because D1 snaps to a *sentence* end, the gap to the next word is usually larger than
mid-sentence — speakers pause at full stops. Execution 525's 209 ms is a fast turnaround;
400–800 ms is typical, so the full 400 ms will often be available.

Record `moment.ending.tail_pad_ms` and `moment.ending.tail_pad_clamped` (true when the clamp
reduced it below `TAIL_PAD_MS`). Do **not** throw when the tail clamps to 0 — a host who cuts in
instantly is an acoustic fact, not a fault, and failing a paid run over it would be wrong.

### D5 — Lead-in pad before the first word. **Adopted, at 120 ms.**

```
LEAD_PAD_MS        = 120
PREV_WORD_GUARD_MS = 40

lead = clamp(0, LEAD_PAD_MS, firstWordStart - prevWordEnd - PREV_WORD_GUARD_MS)
lead = min(lead, firstWordStart)        // trim can never go below 0
```

Why add it: the clip currently starts exactly on the first word's detected onset, so the
opening consonant can be clipped. That is the same fault as the ending, at the other end, and
it costs nothing to fix — execution 525 has 948 ms of clear space available.

Why 120 and not 400: at the *start* of a social reel, silence is dead air in the one second
that decides whether the reel is watched at all. 120 ms covers the consonant onset and is
inaudible as a pause. The two ends want different values and should have different constants.

When there is no previous word, `lead = min(LEAD_PAD_MS, firstWordStart)`.

Record `moment.ending.lead_pad_ms` and `moment.ending.lead_pad_clamped`.

### D6 — Hard ceiling on the final duration.

```
MAX_FINAL_DURATION_MS = 59000
budgetMs = Math.max(o.end_ms - o.start_ms, MAX_FINAL_DURATION_MS)
```

Why 59 000 and not 60 000: the probe gate measures the **FFprobe duration of the finished MP4**,
not our contract number. Encoders round to frame boundaries (40 ms at 25 fps) and Shotstack may
add a few milliseconds. Execution 525 asked for 44.985 s and the file measured 45.01 s — **+25 ms**.
1.0 s of margin is forty times the observed error and costs nothing, because the model targets
about 45 s anyway.

Why `budgetMs` takes the **larger** of the model span and 59 s: if the model legitimately
proposes a 59.5 s moment, today it renders. A flat 59 s ceiling would newly reject it. This
change must never make an existing passing case fail. Taking the maximum means the new ceiling
only ever restricts the *extension and the pads*, never the model's own choice.

**When the budget is exceeded, shrink the pads before failing — lead first, then tail.** Leo's
requirement is about the ending, so the tail is the last thing to give up.

```
if (finalMs > budgetMs) reduce lead toward 0
if (finalMs > budgetMs) reduce tail toward 0
if (finalMs > budgetMs) throw MOMENT_FINAL_DURATION_EXCEEDS_CEILING
```

With both pads at 0 the final length equals the model span, which is `<= budgetMs` by
construction. **So `MOMENT_FINAL_DURATION_EXCEEDS_CEILING` should be unreachable.** That is
correct for a defensive assert: if it ever fires, an upstream guard has been changed and the
money guard is the right place to find out. Keep it.

`MAX_FINAL_DURATION_MS` is a named constant in this node and is deliberately **not** read from
the contract's `clip_max_seconds: 60`. The point of the constant is to stay strictly inside that
limit, so it must not be able to drift up to equal it.

### D7 — What `moment.start_ms` / `end_ms` / `duration_seconds` report. **The final clip bounds.**

| Field | New value |
|---|---|
| `moment.start_ms` | final clip start = `firstWordStart - lead` |
| `moment.end_ms` | final clip end = `finalLastWordEnd + tail` |
| `moment.duration_seconds` | `(moment.end_ms - moment.start_ms)/1000` |
| `moment.words` | re-filtered list covering the final span (a superset of the original) |

Four reasons:

1. **The cost guard must price what is actually rendered.** Both cost nodes multiply
   `duration_seconds` by `0.30/60`. If that field held the model's proposal while the render
   used a longer `length`, the ceiling would be computed against a number nobody bought.
2. **The two cost calculations stay identical to 1e-6 automatically**, because both read the
   same `p.moment.duration_seconds` from this same node. Nothing about that needs changing —
   confirmed by reading both nodes. Any design that gave them different numbers would trip
   `SHOTSTACK_ATOMIC_CLAIM_RESULT_MISMATCH`.
3. **The evidence block must be honest.** `Code - Validate Media Probe And Build TEST Evidence`
   prints `selected_moment.duration_seconds` right beside the FFprobe duration it gates on. If
   those describe different things the evidence misleads whoever reads it.
4. **Airtable must be truthful.** `source_start_ms` / `source_end_ms` exist so a human can find
   the clip in the source recording. They must be the clip's real bounds.

**The model's proposal is not discarded — it is the audit trail.** Preserve it as:

```
moment.model_proposed_start_ms
moment.model_proposed_end_ms
moment.model_proposed_duration_seconds
moment.ending = { mode, lead_pad_ms, tail_pad_ms, lead_pad_clamped, tail_pad_clamped,
                  extended_by_ms, words_added, final_last_word_text }
```

and carry the same block into `selection_json`, which is written to Drive as the moment
evidence file. Without it there is no way to tell, after the fact, whether the snap fired.

### D8 — New error identifiers

Existing SCREAMING_SNAKE style. Four added, all in this node.

| Identifier | Fires when |
|---|---|
| `MOMENT_FINAL_DURATION_EXCEEDS_CEILING` | Final length still exceeds the budget after both pads are reduced to 0. Defensive; should be unreachable. |
| `MOMENT_CLIP_BOUNDS_INVALID` | `clipStart < 0`, or `clipEnd <= clipStart`, or `clipStart > firstWordStart`, or `clipEnd < finalLastWordEnd`. Catches a sign error in the pad arithmetic before money is spent. |
| `MOMENT_SENTENCE_SNAP_WORDS_MISSING` | The re-filtered word list is empty, shorter than the original, or its first word is not the original first word. The final list must be a superset. |
| `MOMENT_WORD_INDEX_RANGE_INVALID` | The indexes of the model's words inside the full transcript array are not contiguous, so "the word after the last one" is not well defined. |

**Every existing guard is kept, unchanged and in the same order:**
`OPENAI_RESPONSE_NOT_COMPLETED`, `OPENAI_OUTPUT_TEXT_MISSING`, `OPENAI_OUTPUT_JSON_INVALID`,
`OPENAI_SCHEMA_KEYS_INVALID`, `OPENAI_SCHEMA_TYPES_INVALID`, `MOMENT_DURATION_OUT_OF_BOUNDS`,
`MOMENT_WORD_EVIDENCE_MISSING`, `MOMENT_BOUNDARY_NOT_WORD_ALIGNED`,
`VTT_CAPTION_CONTRACT_INVALID`, `VTT_CUE_GAP_COLLAPSE`.

Note that `VTT_CAPTION_CONTRACT_INVALID` tests `g.end_ms > d*1000 + 100`. With a lead pad, group
end offsets grow by `lead` while `d*1000` grows by `lead + tail`, so the check keeps *more*
margin than today. No adjustment needed.

---

## 3. Order of operations in the edited node

The order matters because the pads and the search depend on each other. Coder must follow it.

1. Parse and validate the OpenAI JSON — **unchanged**.
2. `d0 = (o.end_ms - o.start_ms)/1000`; throw `MOMENT_DURATION_OUT_OF_BOUNDS` if outside 30–60 —
   **unchanged**.
3. Walk `t.words` once, collecting **indexes** of the words inside the model span
   (`start >= o.start_ms && end <= o.end_ms`). Keeping indexes, not just objects, is what makes
   "the previous word" and "the next word" available without a lookup that could fail.
   Empty → `MOMENT_WORD_EVIDENCE_MISSING`. Non-contiguous → `MOMENT_WORD_INDEX_RANGE_INVALID`.
4. `MOMENT_BOUNDARY_NOT_WORD_ALIGNED` against the model's proposal — **unchanged**.
5. Compute `lead` (D5). It depends only on the first word and the word before it, both known
   now, so it can be fixed before the search. `clipStart = firstWordStart - lead`.
6. Compute `budgetMs` (D6).
7. If the model's last word already ends a sentence → `mode = already_sentence_end`, no search.
   Otherwise run the forward search (D1, D2, D3), stopping on speaker change, horizon, budget,
   or transcript end.
8. Re-filter the word list to `start >= o.start_ms && end <= finalLastWordEnd`. Check the
   superset invariant → `MOMENT_SENTENCE_SNAP_WORDS_MISSING`.
9. Compute `tail` (D4). `clipEnd = finalLastWordEnd + tail`.
10. Apply the budget shrink (D6): lead, then tail, then
    `MOMENT_FINAL_DURATION_EXCEEDS_CEILING`.
11. Check `MOMENT_CLIP_BOUNDS_INVALID`.
12. `d = (clipEnd - clipStart)/1000`.
13. Build caption groups of four from the **final** word list, with every offset relative to
    **`clipStart`**, not `o.start_ms`.
14. `VTT_CAPTION_CONTRACT_INVALID`, `GAP_MS = 40`, `displayEnds`, `VTT_CUE_GAP_COLLAPSE`, VTT
    serialisation — **all unchanged**.
15. Shotstack request: source clip `trim: clipStart/1000`, `length: d`, `fit: 'crop'`. TEST badge
    clip `start: 0`, `length: d`. Caption clip `length: 'end'` — **unchanged**.
16. Return, with the moment overrides of D7.

### Three traps Coder must not fall into

1. **`moment: { ...o, ... }` spreads the model's `start_ms` and `end_ms`.** The final bounds must
   be assigned **after** the spread, or the spread silently wins and Airtable, the cost guard
   and the evidence all report the wrong numbers while the render uses the right ones.
2. **Caption offsets must subtract `clipStart`, not `o.start_ms`.** There are four subtractions
   in the group builder (`g[0].start`, `g.at(-1).end`, and both inside the per-word map). All
   four change. Missing one desynchronises the captions from the picture.
3. **The TEST badge clip's `length` is also `d`.** It is easy to change the source clip and
   forget the badge, leaving the badge to vanish 289 ms before the video ends.

### The last caption cue is deliberately NOT extended to the clip end

With a 169 ms tail, the final cue's VTT end (45.105 s) falls before the clip end (45.274 s), so
the caption disappears for the last 169 ms.

That is left alone on purpose. Extending the last cue would change that group's displayed
duration, and if Shotstack spreads the highlight across the displayed duration — the working
hypothesis behind the drift — it would slow the last group's highlight and partially mask the
very fault the A/B diagnostic `YOUR_WORKFLOW_ID_2` is being run to measure. **This design must not
contaminate that measurement.** 169 ms of clean video after the speech has finished is an
honest ending, not a defect. Revisit only after the A/B diagnostic reports.

---

## 4. Worked example — execution 525's real numbers

A known answer for Coder to implement against and QA to check.

**Inputs**
```
o.start_ms = 55754      o.end_ms = 100739      model span = 44985 ms
first word  "Well,"  start 55754      previous word "these?" end 54806
last word   "here."  span 100610-100739       next word "It's" start 100948
source_duration_seconds = 175.999   ->  175999 ms
```

**Step by step**

| Step | Result |
|---|---|
| `d0` | 44.985 s → inside 30–60, passes |
| word alignment | `\|55754-55754\|=0`, `\|100739-100739\|=0` → passes |
| lead | `clamp(0, 120, 55754 - 54806 - 40) = clamp(0,120,908) = **120 ms**`, not clamped |
| `clipStart` | `55754 - 120 = **55634 ms**` |
| `budgetMs` | `max(44985, 59000) = 59000 ms` |
| sentence test on `here.` | strip closers → `here.`; terminator `.`; core `here`, length 4; not an abbreviation; next word `It's` → `I` uppercase → **true** |
| mode | **`already_sentence_end`** — no forward search runs, extension 0 ms, 0 words added |
| `finalLastWordEnd` | 100739 ms (unchanged) |
| word list | 111 words, identical to today. Superset invariant holds trivially |
| tail, next-word clamp | `clamp(0, 400, 100948 - 100739 - 40) = clamp(0,400,169) = **169 ms**`, clamped |
| tail, source clamp | `175999 - 100739 - 100 = 75160` → not binding |
| `clipEnd` | `100739 + 169 = **100908 ms**` |
| final length | `100908 - 55634 = 45274 ms` → `45274 <= 59000` → no shrink |
| bounds check | `55634 >= 0`, `100908 > 55634`, `55634 <= 55754`, `100908 >= 100739` → passes |
| `d` | **45.274 s** |

**Shotstack request**
```
source clip:  trim: 55.634    length: 45.274    fit: 'crop'
TEST badge:   start: 0        length: 45.274
caption clip: start: 0        length: 'end'      (unchanged)
```

**Caption offsets — every existing offset from execution 525 shifts by exactly +120 ms.**
That single invariant is the cheapest thing for QA to check.

| Cue | Today | After |
|---|---|---|
| 1, start | 0 ms | **120 ms** |
| 1, raw end | 1549 ms | 1669 ms |
| 1, displayed end (`GAP_MS = 40`) | 1509 ms | **1629 ms** |
| 1, VTT line | `00:00:00.000 --> 00:00:01.509` | **`00:00:00.120 --> 00:00:01.629`** |
| 2, start | 1549 ms | 1669 ms |
| 28 (last), end of `here.` | 44985 ms | **45105 ms** |
| 28, displayed end (last cue, no gap applied) | 44985 ms | **45105 ms** |

Group count stays **28**. Word count stays **111**. Relative spacing inside every cue is
unchanged, so the highlight-drift measurement is untouched.

**Moment fields reported**
```
moment.start_ms        = 55634
moment.end_ms          = 100908
moment.duration_seconds= 45.274
moment.model_proposed_start_ms = 55754
moment.model_proposed_end_ms   = 100739
moment.model_proposed_duration_seconds = 44.985
moment.ending = { mode: 'already_sentence_end', lead_pad_ms: 120, tail_pad_ms: 169,
                  lead_pad_clamped: false, tail_pad_clamped: true,
                  extended_by_ms: 0, words_added: 0, final_last_word_text: 'here.' }
```

**Projected cost — identical in both cost nodes**
```
assembly_estimate_usd = (175.999/3600) * 0.23      = 0.0112  (toFixed(4))
openai_reserved_usd                                = 0.0500
shotstack part        = 45.274 / 60 * 0.30         = 0.22637
projected             = Number((0.0612 + 0.22637).toFixed(4)) = 0.2876
```
Today's figure is **0.2861**. The change adds **USD 0.0015**. Ceiling is 0.50, so headroom is
0.2124. Even at the 59.0 s worst case the projection is `0.0612 + 0.295 = 0.3562`, still clear.
**The extension can never put the cost ceiling at risk.**

**Predicted FFprobe duration of the finished file:** about **45.30 s** (45.274 s plus the ~25 ms
container rounding measured in 525). Inside 30–60 with 14.7 s to spare.

---

## 5. Constants, in one place

```
LEAD_PAD_MS            = 120     ms   pad before the first word
PREV_WORD_GUARD_MS     = 40      ms   clearance kept from the previous word
TAIL_PAD_MS            = 400     ms   target pad after the last word
NEXT_WORD_GUARD_MS     = 40      ms   clearance kept from the next word's onset
END_OF_MEDIA_GUARD_MS  = 100     ms   clearance kept from the end of the source file
MAX_EXTEND_MS          = 6000    ms   how far forward the sentence search may run
MAX_FINAL_DURATION_MS  = 59000   ms   ceiling on the final clip, 1.0 s inside the probe gate
GAP_MS                 = 40      ms   UNCHANGED, existing, inter-cue gap
```

---

## 6. What this design does NOT change

Stated explicitly so nobody widens the edit. Leo objected to rebuilding when he did not ask
for it.

- No node is added, removed, renamed, moved or rewired. **One node's `jsCode` changes.**
- `GAP_MS = 40`, `displayEnds`, `VTT_CUE_GAP_COLLAPSE`: untouched.
- Caption grouping stays four words. Font stays Arial 52 weight 700. Highlight stays `#FFD400`.
- `fit: 'crop'`: untouched.
- The caption highlight-timing fault is a separate, still-unproven problem owned by
  `YOUR_WORKFLOW_ID_2`. **This design deliberately does not touch cue durations**, and section 3
  explains where it declined to.
- No retry, sub-workflow, human-review point or error route changes. This node's `onError` and
  the error routes around it are unchanged.
- No new external API call. No new cost exposure beyond USD 0.0015 per render.

### Sub-workflow extraction

Considered, rejected. The rule of thumb is to extract anything over roughly 10 nodes or reused
twice. This is inside one existing node and is used once. Extraction would add a call boundary
and a serialisation step for no benefit, and would break the "surgical" constraint.

### Retry strategy

Not applicable and unchanged. This is a pure Code node with no network call, so there is nothing
to retry — a failure here is deterministic and would fail identically on every attempt.
`retryOnFail` must stay `false`.

### Idempotency

Unchanged and already handled upstream of the money by `render_key` plus
`RETIRED_RENDER_KEYS` in `Code - Refuse Duplicate Shotstack Submission And Build Claim`. The
dedupe key is `render_key` = `<test_run_id>|RENDER-nn`, enforced against the Airtable events
row's `payload_hash`. This design does not change that mechanism — but see the companion
changes below, because it does require a new key value.

### Database usage

Unchanged in shape. Airtable base `appYOURBASEID0000`, moments table `tblYOURTABLE00005`, events
table `tblYOURTABLE00002`. What changes is only the *values* written to
`source_start_ms`, `source_end_ms`, `words_json` and `scores_json`, per D7.

### Human-review points

Unchanged. The safe terminal state remains
`TEST_COMPLETE_PENDING_HUMAN_CAPTION_REVIEW`. No new approval gate is added, and none is
removed.

### Error routing

Unchanged. All four new errors are thrown from the same node as the ten existing ones and
follow the same route. No new failure destination is created.

---

## 7. Companion changes — REQUIRED, or this edit has no visible effect

These are not part of the node edit, but the change will silently fail without them. They are
pre-existing project procedure, recorded here so they are not missed.

**A. Re-key the VTT artifact.** The workflow searches Drive by `appProperties` and **reuses a
matching file**. Every caption offset moves by 120 ms, so the VTT content changes. Without a
re-key, the old file (`captions_vtt_v2`) is reused and the new offsets never reach Shotstack —
the fix will look as though it did nothing. Bump `captions_vtt_v2` → `captions_vtt_v3` in all
six places. This trap already cost execution 519.

**B. New render key, with retirement in the same edit.** `RENDER-02` was spent by execution 525
and the guard now blocks. A further paid render needs `RENDER-03`, and QA criterion 26 requires
`RETIRED_RENDER_KEYS` to gain `podcast-runtime-test-20260911-01|RENDER-02` **in the same edit**.
A key bump without a retirement is rejected at QA.

**C. Fresh approval from Leo before the paid render.** Approved test spend was USD 0.50 and
roughly USD 0.5722 has already been projected across two renders. Any further paid render needs
a new yes.

---

## 8. Acceptance checks for QA and Tester

Numbered so Tester can verify and King can compare.

1. The diff touches exactly one node: `Code - Parse Validate Moment Copy And Build VTT Render Contract`. No connection, position or node-list change.
2. All ten existing error identifiers are still present and in the same order.
3. `GAP_MS = 40` and the `VTT_CUE_GAP_COLLAPSE` fallback are byte-identical to today.
4. `fit: 'crop'` is still on the source clip.
5. The eight constants of section 5 appear as named constants with the stated values.
6. `moment.start_ms` / `end_ms` / `duration_seconds` are assigned **after** the `...o` spread.
7. Re-running execution 525's stored inputs yields `mode: 'already_sentence_end'`, lead 120 ms, tail 169 ms, `trim 55.634`, `length 45.274`, 28 groups, 111 words.
8. Cue 1's VTT line reads `00:00:00.120 --> 00:00:01.629`.
9. The last cue's displayed end is 45.105 s, and the last cue is **not** stretched to the clip end.
10. Every caption offset equals its execution-525 value plus exactly 120 ms.
11. Both cost nodes compute `0.2876` and agree to `< 0.000001`.
12. `moment.ending` and the three `model_proposed_*` fields are present in the node output and in `selection_json`.
13. After the render, FFprobe duration is between 45.20 and 45.40 s, and inside 30–60.
14. After the render, audio RMS across the final 100 ms is below 568 — the value measured at 44.95 s in 525 — confirming the tail is decay and silence, not speech.
15. Frame at 45.20 s shows video with no caption and no other speaker's voice.

---

## 9. Open questions Leo or the Orchestrator must answer

1. **Only one.** `TAIL_PAD_MS = 400` is the target, but execution 525 clamps to 169 ms because
   the other host starts speaking 209 ms later. **Is 169 ms of silence an acceptable ending for
   Leo, or does he want the reel to stop slightly earlier — at the previous sentence end — when
   the full 400 ms is not available?** This design says 169 ms is enough and does not add that
   behaviour. Changing it later is a one-line addition.

Flagged, not a question, but the Orchestrator should note it:

- **A pre-existing hazard this design does not fix.** A model span of exactly 60.000 s passes
  `MOMENT_DURATION_OUT_OF_BOUNDS` today, and the finished MP4 would then measure about 60.02 s
  and **fail `duration_in_contract` after the render is paid for**. That risk exists now and is
  unchanged by this design. Fixing it means tightening `MOMENT_DURATION_OUT_OF_BOUNDS`, which is
  outside the surgical scope Leo set. Worth a separate, tiny change order.

---

## 10. What the Integrator must verify

One item, and it is decisive.

1. **AssemblyAI must be returning punctuated, diarised words for the real 47-minute podcast.**
   The whole design rests on two properties of `transcript.words[]`:
   - `word.text` carries sentence punctuation (`punctuate` enabled). If punctuation is off,
     `isSentenceEnd` can never return true, every run falls back, and the feature silently does
     nothing — visible only through `moment.ending.mode`.
   - `word.speaker` is populated (`speaker_labels` enabled). If it is always `null`, D3's
     speaker-change stop becomes inert and the reel can run into the other host.

   Verify both against the **stored** result of job `00000000-0000-0000-0000-000000000000` —
   reading it is free. **Never resubmit that job.**

2. Nothing else. No new endpoint, body, header, rate limit, pagination path or error code is
   introduced. Shotstack's `trim` and `length` already accept three-decimal seconds — proven by
   execution 525 rendering `55.754` / `44.985` successfully.
