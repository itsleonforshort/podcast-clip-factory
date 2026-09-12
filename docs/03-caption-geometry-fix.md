# Podcast VTT A/B Diagnostic Geometry Fix — Architecture v1

**Scope:** One surgical correction in workflow `YOUR_WORKFLOW_ID_2`, `Podcast VTT Caption Check - SANDBOX - INACTIVE`.

**Decision:** Coder changes only `Code - Build AB Diagnostic Render Request` (`n8n-nodes-base.code`, typeVersion 2). No node, connection, credential, timing, font, VTT, source-video, poll, or workflow-setting change is part of this fix.

## Exact correction

| Track A element | Current | Required | Reason |
|---|---:|---:|---|
| `A cue-level` label, `position:'top'` | `offset:{x:0,y:0.04}` | `offset:{x:0,y:-0.04}` | Shotstack positive Y moves upward. The negative value moves the 60 px label into the 1080x1920 frame and matches the proven direction used by the main workflow's top badge. |
| Cue-level rich caption, `position:'top'` | `offset:{x:0,y:0.14}` | `offset:{x:0,y:-0.14}` | The sign inversion keeps the established spacing magnitude while moving the complete 360 px caption box down from the top edge. |

Track B stays exactly as built: label `position:'bottom', y:0.34`; caption `position:'bottom', y:0.12`. The source clip remains `fit:'crop'`.

## Existing node sequence and boundaries

No sub-workflow applies. This is a bounded 12-node manual diagnostic and extracting one geometry edit would add failure surface without reuse.

1. `Manual Trigger - Start VTT Diagnostic` — `n8n-nodes-base.manualTrigger`
2. `Code - Build AB Word Timing Cues From Execution 525 Evidence` — `n8n-nodes-base.code`
3. `Google Drive - Create Track A Cue-Level VTT` — `n8n-nodes-base.googleDrive`
4. `Google Drive - Create Track B Per-Word VTT` — `n8n-nodes-base.googleDrive`
5. `Code - Build AB Diagnostic Render Request` — `n8n-nodes-base.code`; apply only the two offset changes above
6. `HTTP Request - Submit VTT Sandbox Diagnostic` — `n8n-nodes-base.httpRequest`; the only render submission
7. `Code - Preserve Diagnostic Render ID` — `n8n-nodes-base.code`
8. `Wait - Pause Before Diagnostic Status` — `n8n-nodes-base.wait`
9. `HTTP Request - Read Diagnostic Render Status` — `n8n-nodes-base.httpRequest`
10. `Code - Classify Bounded Diagnostic Status` — `n8n-nodes-base.code`
11. `If - Continue Diagnostic Polling` — `n8n-nodes-base.if`; true returns to step 8 and false advances
12. `Code - Return Diagnostic Evidence` — `n8n-nodes-base.code`

## Reliability and control contract

- **Retries and backoff:** Keep the existing bounded status loop unchanged: wait between reads and stop after at most 12 status reads. Never route any status path back to the render POST. Node-level submission retry remains off because a transport timeout can hide a successful paid submission.
- **Idempotency:** No persistent dedupe guard exists in this disposable manual diagnostic. The logical key is the existing `diagnostic_key`, but it is evidence metadata rather than an enforced lock. Control the exposure by one supervised click. A second click is prohibited for this test because it would create two more Drive files and another render.
- **Stored data and keys:** Execution data must retain `diagnostic_key`, both VTT file IDs and public URLs, the Shotstack render ID, poll count/status, and final evidence URL or provider failure. The two VTT artifacts remain in the existing test Drive folder. No new store or schema applies.
- **Human review:** After the single render completes, Tester inspects Track A and Track B at execution 525's known word boundaries. Human review must confirm the upper label and complete Track A caption box remain visible before judging word-highlight timing. No publishing or production approval follows from this diagnostic.
- **Notifications:** Not applicable. The manual supervised run is watched in n8n and its execution record is the notification surface. Adding email or chat would create unrelated external writes.
- **Paid-call exposure:** Exactly one 10-second Shotstack stage render may occur. Treat sandbox usage as billable because billing is unverified. Google Drive creates exactly two VTT files. The change itself makes no provider call.

## Error routes

- Missing Track A or Track B Drive file ID throws before Shotstack and stops the workflow.
- Either Drive create failure stops before Shotstack.
- Render submission failure stops with no poll. Do not retry the POST automatically.
- Missing render ID after a successful response throws immediately.
- A status-read network failure stops the run. Recover the preserved render ID from execution data and inspect that job; never rerun the POST to recover it.
- `queued`, `fetching`, `generating`, `rendering`, `saving`, and other non-terminal states remain inside the capped wait loop. `done` or provider failure exits to the evidence node. Poll exhaustion returns bounded failure evidence and must not resubmit.

## Acceptance criteria before one supervised run

1. Live and exported code show Track A label `y:-0.04` and Track A rich caption `y:-0.14`.
2. Track B offsets, 1080x1920 output, ten-second duration, caption styles, VTT contents, and `fit:'crop'` are byte-for-byte unchanged apart from the two literals.
3. Workflow validation reports zero errors and zero invalid connections. The workflow stays inactive and unavailable in MCP.
4. QA retrieves the live node after Coder's update and confirms the exact two-value diff.

No other correction is required before one supervised run. The run itself remains necessary because inline per-word WebVTT timing is an unresolved provider behavior and cannot be approved from schema or static validation.

## Template and node evidence

n8n template search found Shotstack templates 4630 by Immanuel and 11724 by Alexandru Burca. Neither was used because both describe broader production and publishing flows and provide no evidence for this A/B overlay geometry. Live `get_node` confirms the changed node is `nodes-base.code` version 2 with JavaScript in all-items mode. Live workflow retrieval confirms the node and the 12-node sequence above.

## Integrator verification

No new external contract needs verification for this geometry-only patch. Existing Shotstack evidence already establishes positive Y as upward and the main workflow has proven a negative top offset. The only open Shotstack question is whether inline per-word timestamps are honored; the controlled A/B render is the verification. Google Drive behavior and credentials are unchanged.
