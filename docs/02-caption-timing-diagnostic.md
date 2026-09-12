# Podcast VTT diagnostic architecture v1

Date: 2026-09-12. Owner: Designer. Status: design complete for Coder implementation; not QA, execution, or deployment approval.

## Purpose and boundary

One manually initiated 4.2-second Shotstack sandbox diagnostic checks whether the exact existing Drive VTT is accepted and whether highlighting advances within each four-word cue. Use the literal request in `docs/reports/podcast-integrator-vtt-preflight-522.md`, section **Exact diagnostic request for Tester**, without changing any body field. This is a standalone new inactive workflow. Keep MCP exposure off. Do not change main workflow `YOUR_WORKFLOW_ID`, the existing status checker, or the RENDER-02 claim.

The user authorized the described correction and test with “Do it.” QA must inspect the built diagnostic before Tester or the supervised manual click submits its single request. The diagnostic does not establish full-reel quality or audio alignment by listening because it is silent.

## Node sequence

| # | Exact name | Exact node type | Behavior and route |
|---|---|---|---|
| 1 | Manual Trigger - Start VTT Diagnostic | `n8n-nodes-base.manualTrigger` | One item, one supervised execution. Go to 2. |
| 2 | HTTP Request - Submit VTT Sandbox Diagnostic | `n8n-nodes-base.httpRequest` | POST `https://api.shotstack.io/edit/stage/render`. Exact literal Integrator JSON. Existing Shotstack API Header Auth credential; bare `x-api-key` supplied by credentials. JSON content type. No query parameters. Timeout 30 seconds. Automatic retries disabled. Go to 3. |
| 3 | Code - Preserve Diagnostic Render ID | `n8n-nodes-base.code` | Normalize the actual JSON response wrapper, require success and a nonempty `response.id`, preserve the raw sanitized response and ID. Initialize `poll_count=0`, `started_at`, `diagnostic_key`, and `next_wait_seconds=10`. Missing ID or invalid acceptance throws a structured `SUBMISSION_UNRESOLVED` error. Never resubmit. Go to 4. |
| 4 | Wait - Pause Before Diagnostic Status | `n8n-nodes-base.wait` | Time interval of `next_wait_seconds`; normally 10 seconds. Carry the complete current context. Go to 5. |
| 5 | HTTP Request - Read Diagnostic Render Status | `n8n-nodes-base.httpRequest` | GET `https://api.shotstack.io/edit/stage/render/{render_id}` with the same existing credential. Timeout 30 seconds. Automatic retries disabled. Preserve the input context through paired input from node 4 and capture status/body. Go to 6. |
| 6 | Code - Classify Bounded Diagnostic Status | `n8n-nodes-base.code` | Recover the current loop item, increment `poll_count` once per GET, retain ID and provider response. `done` requires a nonempty HTTPS `response.url`, then `pending=false`. `failed` throws `RENDER_FAILED`. A valid nonempty unknown intermediate status, queued/rendering status, or retryable read response can remain pending only while `poll_count<12`. At limit throw `POLL_LIMIT_REACHED` with ID; the provider job may still exist. Missing/malformed status or mismatched returned ID throws `STATUS_CONTRACT_ERROR`. Go to 7 for valid outcomes. |
| 7 | If - Continue Diagnostic Polling | `n8n-nodes-base.if` | Test Boolean `pending` strictly. True loops only to 4. False goes only to 8. No route returns to POST. |
| 8 | Code - Return Diagnostic Evidence | `n8n-nodes-base.code` | Assert done plus ID and HTTPS URL. Return `diagnostic_key`, `render_id`, `status`, `url`, `poll_count`, and sanitized terminal response for Tester download and review. End. Invalid terminal data throws. |

Code nodes are justified by response normalization, preserving loop identity, and bounded provider-state classification; these checks are more involved than a field assignment. One sub-workflow is not needed: this is an eight-node standalone diagnostic with one caller and no shared write logic.

## Failures, retries, and evidence

POST has exactly one attempt. Any transport failure, non-success HTTP response, malformed acceptance, or missing render ID stops execution. A timeout or 5xx can hide an accepted job: retain the execution and reconcile it before another submission. Do not retry the whole workflow automatically.

GET captures full HTTP status and body so node 6 can classify HTTP failures. HTTP 429 waits 60 seconds before the next bounded GET. HTTP 5xx waits 20 then 40 then at most 60 seconds on consecutive failures; successful pending responses reset the wait to 10 seconds. These reads still count against the same 12-GET cap. Auth errors and other non-retryable 4xx stop immediately. A network exception stops n8n execution with the preserved ID available in node 3 and the prior loop item; it does not trigger another POST. Code, Wait, or expression exceptions stop execution visibly. Save successful and failed execution data. Set overall execution timeout to 900 seconds. No infinite loop or silent error continuation.

No separate Error Trigger or notification service is required because this is a supervised manual diagnostic. The execution result is the notification surface; Tester reports the ID and exact failure to the user. No email, Slack, or other messages are sent.

## Idempotency and stored keys

Use `diagnostic_key=podcast-vtt-522-v1-4p2s` for reconciliation. Store this key, request body, execution ID, returned render ID, polling history, and result URL in n8n execution evidence and the Tester report. No Airtable or Drive writes. No durable atomic dedupe is claimed: the one-click supervised boundary and absence of a loop back to POST prevent within-run duplicate submissions. Before the authorized click, Tester checks execution history for this exact diagnostic key; if an ID already exists, reuse it through read-only inspection rather than execute this submission workflow again. RENDER-02 remains untouched.

## Cost and review boundary

One ordinary sandbox render is expected to add zero rendering cost according to the current official documentation verified by Integrator. Sandbox account-credit eligibility still applies. No AI, speech generation, auto-transcription, image generation, production endpoint, or production credential is permitted. No existing USD ceiling field changes. The exact request is rich-caption from the saved VTT on a plain background at 640x360 and 30 fps for 4.2 seconds. A different body or endpoint requires another review; no fallback service is designed here.

Tester must inspect the downloaded MP4 at multiple frames inside each cue and around 1.549 seconds. Visible words must match the VTT. Lines must stay within four words. A yellow active word must advance individually. Compare observed change times against existing AssemblyAI word evidence relative to 55.754 seconds, using the existing QA/PRD tolerance. `done` alone does not pass B2. A full-reel run remains a separate stage after B1 and B2 pass.

## Discovery and external contracts

Searched n8n templates for Shotstack before this design. Results were templates 4630 and 11724; neither is reused because both add unrelated content-generation/publishing scope. No template attribution is required for reused content because none was copied. MCP `get_node` checked Manual Trigger, HTTP Request, Code, Wait, If, and Stop and Error; the final minimal design does not need the last type. Coder must use supported node versions and inspect HTTP full-response parsing against the live schema.

Integrator's cited report supplies the exact submit body, credential scheme, acceptance ID shape, status URL shape, sandbox limits, and billing distinction. Runtime still needs to establish Drive subtitle detection and within-cue highlighting/alignment. No additional design-blocking question remains. This document completes the bounded architecture step; independent QA and Tester still own their respective gates.
