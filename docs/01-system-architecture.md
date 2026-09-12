# Podcast Video Automation Architecture v1

**Date:** 2026-09-11  
**Status:** Design only. No workflow JSON, deployment, activation, or paid test is authorized by this document.  
**Requirements source:** `podcast-video-automation-prd.md` v1.0. The attachment is treated as product requirements. Repository safety and naming rules still apply.

## 1. Build order and decision gate

Build the **caption technical spike first**. Do not build the seven-workflow production system until the spike produces evidence that Shotstack can meet all six caption requirements. If any requirement fails, keep the provider-neutral render contract below and send the same contract to a Remotion plus FFmpeg service instead.

The production system contains the seven workflows named in the PRD. Workflow 1 is the intake orchestrator. Workflows 2 through 6 are event-driven workers. Workflow 7 is the shared error recorder and notifier. All workflows are created inactive. Leo activates them only after QA, testing, and King approval.

Template research found no complete template matching this system. Two templates informed the design without being imported: n8n template **11724**, “Generate News Digest Videos from WordPress to YouTube with Shotstack,” by Alexandru Burca, supplied the submit, wait, status, and download pattern. Template **4887**, “Auto-Create Podcast from YouTube Transcript using Dumpling AI and GPT-4o,” by Yang, supplied the transcript, structured AI, and Airtable pattern. This architecture adds bounded polling, version locks, cost gates, source validation, and full failure routes that those examples do not provide.

## 2. Verified n8n node catalogue

Every proposed node type was checked with n8n-MCP `get_node` on 2026-09-11.

| Short type | Workflow type | Current version |
| --- | --- | --- |
| `nodes-base.manualTrigger` | `n8n-nodes-base.manualTrigger` | 1 |
| `nodes-base.googleDriveTrigger` | `n8n-nodes-base.googleDriveTrigger` | 1 |
| `nodes-base.webhook` | `n8n-nodes-base.webhook` | 2.1 |
| `nodes-base.respondToWebhook` | `n8n-nodes-base.respondToWebhook` | 1.5 |
| `nodes-base.set` | `n8n-nodes-base.set` | 3.5 |
| `nodes-base.code` | `n8n-nodes-base.code` | 2 |
| `nodes-base.if` | `n8n-nodes-base.if` | 2.3 |
| `nodes-base.switch` | `n8n-nodes-base.switch` | 3.4 |
| `nodes-base.httpRequest` | `n8n-nodes-base.httpRequest` | 4.5 |
| `nodes-base.wait` | `n8n-nodes-base.wait` | 1.1 |
| `nodes-base.googleDrive` | `n8n-nodes-base.googleDrive` | 3 |
| `nodes-base.airtable` | `n8n-nodes-base.airtable` | 2.2 |
| `nodes-base.airtableTrigger` | `n8n-nodes-base.airtableTrigger` | 1 |
| `nodes-base.executeWorkflowTrigger` | `n8n-nodes-base.executeWorkflowTrigger` | 1.2 |
| `nodes-base.executeWorkflow` | `n8n-nodes-base.executeWorkflow` | 1.3 |
| `nodes-base.splitInBatches` | `n8n-nodes-base.splitInBatches` | 3 |
| `nodes-base.errorTrigger` | `n8n-nodes-base.errorTrigger` | 1 |
| `nodes-base.scheduleTrigger` | `n8n-nodes-base.scheduleTrigger` | 1.4 |
| `nodes-base.crypto` | `n8n-nodes-base.crypto` | 2 |

No AI Agent node is used. Transcript analysis and writing are one-shot schema-controlled calls to the OpenAI Responses API. Tool autonomy and conversational memory do not apply.

## 3. Caption technical spike

### Contract

Input is one 20–30 second source range plus its original word array. Required fields are `source_url`, `source_start_ms`, `source_end_ms`, `words[]`, `caption_style`, `output_profile`, `allow_render`, and `test_mode`. Each word contains `text`, `speaker`, `start_ms`, `end_ms`, and `confidence`.

The provider-neutral output is `render_contract` with the trimmed source range, output profile, caption groups, word-relative timing, payload hash, and expected duration. Each group has no more than eight words. Each line has no more than four. Only one word may have an active interval at any instant. Every relative time is calculated as `original_ms - source_start_ms` and must stay between zero and the trimmed clip duration.

The grouping code prefers punctuation and pauses of at least 350 ms. It avoids separating a determiner from its noun and avoids breaking short names. A forced break occurs before word nine. Intervals must be monotonic and non-overlapping. A group is hidden when there is no active speech.

### Exact node sequence

1. `Manual Trigger - Start Caption Spike` (`n8n-nodes-base.manualTrigger`).
2. `Edit Fields - Load Caption Spike Fixture` (`n8n-nodes-base.set`). Loads fictional timing data and configuration. `allow_render` defaults to false.
3. `Code - Convert Source Times And Group Captions` (`n8n-nodes-base.code`). Produces the provider-neutral contract.
4. `Code - Assert Caption Manifest Contract` (`n8n-nodes-base.code`). Checks line limits, active-word overlap, timing bounds, phrase grouping, and trim-relative arithmetic.
5. `IF - Is Caption Manifest Valid?` (`n8n-nodes-base.if`). False routes to `Edit Fields - Return Caption Contract Failure` and stops.
6. `Crypto - Hash Caption Render Contract` (`n8n-nodes-base.crypto`). SHA-256 of canonical contract JSON.
7. `IF - Is Shotstack Render Allowed?` (`n8n-nodes-base.if`). False routes to `Edit Fields - Return Zero-Cost Caption Evidence` and stops.
8. `HTTP Request - Submit Shotstack Caption Render` (`n8n-nodes-base.httpRequest`). Disabled when first built. One submission attempt only.
9. `Airtable - Save Shotstack Spike Job ID` (`n8n-nodes-base.airtable`). Saves the job ID before any wait.
10. `Wait - Pause Before Shotstack Status Check` (`n8n-nodes-base.wait`).
11. `HTTP Request - Read Shotstack Render Status` (`n8n-nodes-base.httpRequest`).
12. `Switch - Route Shotstack Render State` (`n8n-nodes-base.switch`). `done`, `failed`, `queued/rendering`, and fallback outputs are all wired.
13. `Code - Calculate Next Shotstack Poll` (`n8n-nodes-base.code`). Calculates 10, 20, 40, then 60 second waits and total elapsed time.
14. `IF - Is Shotstack Poll Budget Remaining?` (`n8n-nodes-base.if`). True goes through `Wait - Back Off Before Shotstack Poll Retry` and returns to node 11. False records timeout and stops. Maximum elapsed time is 20 minutes.
15. `HTTP Request - Download Shotstack MP4` (`n8n-nodes-base.httpRequest`). Response format is file in `$binary.data`.
16. `Crypto - Hash Downloaded Caption Spike MP4` (`n8n-nodes-base.crypto`). SHA-256 over binary data.
17. `Google Drive - Upload Caption Spike MP4` (`n8n-nodes-base.googleDrive`). Uploads to the configured test output folder.
18. `Google Drive - Verify Caption Spike MP4` (`n8n-nodes-base.googleDrive`). Reads the returned file ID.
19. `Code - Build Caption Proof Evidence` (`n8n-nodes-base.code`). Returns job ID, payload hash, MP4 checksum, Drive file ID, downloadable/preview path, duration, and assertion results.
20. `Airtable - Save Caption Spike Result` (`n8n-nodes-base.airtable`).
21. `Edit Fields - Return Caption Spike Result` (`n8n-nodes-base.set`).

All external nodes use `onError: continueErrorOutput` with the error output wired. Submit failure is not retried because an ambiguous response may already have created a paid job. Status, download, Airtable, and Drive read errors may retry three times in production with 5, 15, and 45 second waits. Test mode permits one attempt. Exhaustion records `SPIKE_FAILED`; it never silently passes.

### Spike decision evidence

Shotstack passes only when one downloadable MP4 proves: one active highlighted word; the prior word is white; no line exceeds four words; phrase groups are natural; the first word’s relative time equals its source time minus the trim start within 100 ms; and the file is downloaded through n8n and verified in Drive. Save the render request, timing manifest, hashes, execution ID, and reviewer observations. If any item fails, Integrator specifies the Remotion/FFmpeg adapter and Coder leaves Shotstack out of Workflow 5.

## 4. Shared identity, storage, retry, and cost rules

### Idempotency keys

- `episode_dedupe_key = sha256("episode|" + source_file_id + "|" + source_version + "|" + source_checksum)`.
- `stage_run_key = sha256(episode_id + "|" + stage + "|" + asset_id_or_none + "|" + asset_version + "|" + input_hash)`.
- `moment_id = sha256(episode_id + "|moment|" + source_start_ms + "|" + source_end_ms + "|" + speaker)`.
- `asset_id = sha256(moment_id + "|asset")`; `asset_version` starts at 1.
- `render_key = sha256(asset_id + "|" + asset_version + "|" + render_purpose + "|" + payload_hash)`.
- `review_decision_key = sha256(review_id + "|" + asset_id + "|" + asset_version + "|" + decision + "|" + decision_modified_at)`.
- `delivery_key = sha256(asset_id + "|" + asset_version + "|final-drive")`.

Before any external write or paid submit, query `Processing Events` by the relevant key. Reuse a stored provider job ID or completed result. Never submit again merely because an HTTP response was lost. Airtable atomic upsert support must be verified by Integrator; a search-then-create sequence is not sufficient under simultaneous triggers.

### Stored records

Airtable tables are `Automation Config`, `Brand Profiles`, `Episodes`, `Moments`, `Assets`, `Reviews`, and `Processing Events`. The PRD fields remain. Add the keys above, `last_completed_stage`, `input_hash`, `payload_hash`, `preview_drive_file_id`, `final_drive_file_id`, `content_package_file_id`, `notification_status`, `retryable`, `token_usage`, and `actual_or_estimated_cost`.

Large transcripts, normalized word JSON, caption manifests, render contracts, and provider response snapshots are files in the configured working Drive folder. Airtable stores Drive file IDs and checksums rather than full media or long transcript bodies.

### Retry policy

- Test mode: one attempt for every external operation.
- Production safe reads and status polls: three attempts, waiting 5, 15, then 45 seconds.
- Airtable and Drive writes: three attempts only after checking the deterministic key or returned file ID.
- AssemblyAI, OpenAI, and Shotstack submissions: one attempt unless Integrator proves a provider idempotency mechanism. An ambiguous submit is `RECONCILIATION_REQUIRED`, not an automatic retry.
- Validation errors, unsupported input, empty AI output, invalid source evidence, and provider 4xx responses are not retried without changing the request.
- Shotstack polling uses 10, 20, 40, then 60 second intervals and stops after 20 minutes. Transcription uses callbacks first; a scheduled recovery scan checks stale jobs without resubmitting them.

### Cost controls

`Automation Config` starts with `test_mode=true`, `allow_transcription=false`, `allow_openai=false`, `allow_render=false`, `test_candidate_limit=3`, `test_asset_limit=1`, and `test_retry_limit=1`. Each paid node has an immediately preceding IF gate. The workflow records expected cost before the call and actual usage after it. A per-episode budget gate holds work at the last completed stage and notifies the owner. No publish node exists.

## 5. Seven workflow contracts

### Workflow 1: Episode intake

**Inputs:** Drive file event or webhook body containing an accessible file reference, source version, brand profile, audience, targets, and candidate count.  
**Output:** `{ok, episode_id, episode_dedupe_key, status, transcription_job_id|null, next_stage}`.

Node sequence:

1. `Google Drive Trigger - Watch Podcast Intake Folder` (`googleDriveTrigger`) and `Webhook - Receive Podcast Intake` (`webhook`) are separate entry points.
2. `Edit Fields - Normalize Drive Intake` / `Edit Fields - Normalize Webhook Intake` (`set`).
3. `Google Drive - Read Source File Metadata` or `HTTP Request - Read Webhook Source Metadata`.
4. `Wait - Confirm Source Upload Is Stable`, then repeat the applicable metadata read.
5. `Code - Validate Source And Configuration`.
6. `IF - Is Intake Valid?`. Invalid webhook input goes to `Respond to Webhook - Reject Invalid Intake` with 400. Drive invalid input creates a failure event. Both stop.
7. `Respond to Webhook - Accept Valid Intake` returns 202 on the webhook branch; processing continues.
8. `Crypto - Build Episode Dedupe Key`.
9. `HTTP Request - Upsert Episode By Dedupe Key` uses Airtable REST atomic upsert if verified.
10. `Airtable - Load Automation Configuration` and `Airtable - Load Brand Profile`.
11. `Switch - Route Existing Episode State`. Already completed or already in progress returns the stored result. `RECEIVED` continues. Fallback fails closed.
12. `IF - Is Transcription Spend Allowed?`. False records a held event and returns.
13. `Switch - Choose Transcription Media Route`. Direct signed URL submits directly. Private Drive media goes through `Google Drive - Download Source Recording` then `HTTP Request - Upload Source To AssemblyAI`.
14. `HTTP Request - Submit AssemblyAI Transcription`.
15. `Airtable - Save Transcription Job And State` sets `TRANSCRIBING` and saves the provider ID immediately.
16. `Edit Fields - Return Episode Intake Result`.

Every metadata read, upload, upsert, and submit error routes to Workflow 7. An ambiguous transcription submit is held for reconciliation and never resubmitted automatically.

### Workflow 2: Transcription completion

**Inputs:** AssemblyAI callback or a scheduled stale-job recovery item.  
**Output:** `{ok, episode_id, transcript_file_id|null, status, next_stage}`.

Node sequence:

1. `Webhook - Receive AssemblyAI Completion` and `Schedule Trigger - Recover Stalled Transcriptions`.
2. `Code - Validate AssemblyAI Callback` followed by `Respond to Webhook - Acknowledge AssemblyAI Callback` with 202. Invalid signatures return 401/400 and stop.
3. Recovery branch uses `Airtable - Find Stale Transcription Jobs` and `Loop Over Items - Check Each Stale Transcription`.
4. Both branches reach `HTTP Request - Read AssemblyAI Transcript Status`.
5. `Switch - Route Transcription State`. Processing stops safely. Failed records `FAILED`. Completed continues. Fallback routes to Workflow 7.
6. `HTTP Request - Download Complete AssemblyAI Transcript`.
7. `Code - Normalize Speakers Words And Confidence`.
8. `Code - Validate Normalized Transcript`.
9. `IF - Is Complete Transcript Valid?`. False records failure without analysis.
10. `Google Drive - Save Normalized Transcript JSON`.
11. `Google Drive - Verify Normalized Transcript JSON`.
12. `Airtable - Mark Episode Transcribed` stores file ID, checksum, and `last_completed_stage=TRANSCRIBED`.
13. `Execute Sub-workflow - Start Transcript Analysis` (`executeWorkflow`), mode once and fire-and-forget after persistence.
14. `Edit Fields - Return Transcription Completion Result`.

Callback duplicates use `stage_run_key`. Recovery polls existing job IDs only. It never creates another transcription job.

### Workflow 3: Transcript analysis

**Typed input:** `episode_id:string`, `stage_run_key:string`.  
**Output:** `{ok, episode_id, selected_moment_ids:array, context_review_ids:array, status}`.

Node sequence:

1. `Execute Workflow Trigger - Receive Transcript Analysis Request` with Define Below inputs.
2. `Airtable - Load Episode For Analysis`, `Airtable - Load Brand Profile For Analysis`, and `Google Drive - Download Normalized Transcript`.
3. `Code - Parse And Chunk Transcript With Overlap` preserves original word indexes and timestamps.
4. `Loop Over Items - Analyze Each Transcript Chunk`, batch size 1.
5. `IF - Is Analysis Spend Allowed?`.
6. `HTTP Request - Generate Structured Chunk Candidates` calls OpenAI Responses with a strict JSON schema.
7. `Code - Validate Chunk Candidate Schema And Evidence`; its output loops back to node 4.
8. Done output reaches `Code - Combine And Remove Repeated Candidates`.
9. `HTTP Request - Compare Structured Candidate Set` performs the final schema-controlled ranking.
10. `Code - Verify Excerpts Timestamps And Word Boundaries` rejects invented text and invalid ranges.
11. `IF - Are Any Candidates Valid?`. False records `FAILED` with `EMPTY_VALID_CANDIDATES`.
12. `Airtable - Upsert Selected Moment Records`.
13. `Switch - Route Moment Context Risk`. High-risk moments create `Airtable - Create Pre-Render Context Review`; safe moments continue.
14. `Execute Sub-workflow - Start Asset Generation`, mode each and fire-and-forget for safe moments.
15. `Airtable - Mark Episode Moments Selected`.
16. `Edit Fields - Return Transcript Analysis Result`.

Transcript text is quoted data inside the prompt and is never treated as workflow instruction. Empty or malformed AI output is an error. A schema correction may occur once only with the validation errors added to a changed request.

### Workflow 4: Asset generation

**Typed input:** `episode_id:string`, `moment_id:string`, `asset_version:number`, `change_scope:string`, `stage_run_key:string`.  
**Output:** `{ok, asset_id, asset_version, payload_hash, render_dispatched, status}`.

Node sequence:

1. `Execute Workflow Trigger - Receive Asset Generation Request`.
2. `Airtable - Load Episode Moment And Brand Context`.
3. `Crypto - Build Asset Version Key`.
4. `Airtable - Find Existing Asset Version`.
5. `IF - Is Asset Version Already Complete?`. True returns the stored result.
6. `IF - Is Writing Spend Allowed?`.
7. `HTTP Request - Generate Structured Written Assets` calls OpenAI Responses.
8. `Code - Validate Quotes Claims And Required Copy`.
9. `IF - Is Written Content Supported?`. False creates a review warning and stops before render.
10. `Code - Build Natural Caption Groups` uses the same tested algorithm as the spike.
11. `Code - Build Provider-Neutral Render Contract`.
12. `Code - Validate Render Contract`.
13. `Crypto - Hash Render Contract`.
14. `Google Drive - Save Caption Manifest And Render Contract`.
15. `Airtable - Upsert Draft Asset Version`.
16. `Execute Sub-workflow - Start Video Rendering And QA`, mode once and fire-and-forget.
17. `Edit Fields - Return Asset Generation Result`.

Any change to copy, timing, captions, branding, or video creates a new asset version. Copy-only revisions may reuse the reviewed video checksum but still require review of the new exact version.

### Workflow 5: Video rendering and QA

**Typed input:** `asset_id:string`, `asset_version:number`, `render_purpose:string`, `payload_hash:string`, `stage_run_key:string`.  
**Output:** `{ok, asset_id, asset_version, qa_status, working_drive_file_id|null, render_key}`.

Node sequence:

1. `Execute Workflow Trigger - Receive Render And QA Request`.
2. `Airtable - Load Asset Render Contract` and `Google Drive - Download Render Contract`.
3. `Crypto - Build Render Key`; `Airtable - Find Existing Render By Key` returns completed work when present.
4. `Code - Adapt Render Contract To Approved Renderer` builds either the proven Shotstack request or the fallback service request.
5. `Code - Validate Provider Render Payload`.
6. `IF - Is Render Spend Allowed?`.
7. `HTTP Request - Submit Approved Video Render`. One submit attempt.
8. `Airtable - Save Render Job ID Immediately` and set `RENDERING`.
9. `Wait - Pause Before Render Status Check`.
10. `HTTP Request - Read Video Render Status`.
11. `Switch - Route Video Render State` with done, failed, processing, and fallback routes.
12. Processing passes through `Code - Calculate Next Render Poll`, `IF - Is Render Poll Budget Remaining?`, and `Wait - Back Off Before Render Poll Retry` before returning to node 10.
13. Done reaches `HTTP Request - Download Rendered MP4` as `$binary.data`.
14. `Crypto - Hash Rendered MP4`.
15. `Google Drive - Upload Working Video Version`.
16. `Google Drive - Verify Working Video Version`.
17. `HTTP Request - Probe Rendered Video` calls the approved media-probe service for playability, duration, codec, dimensions, and audio presence.
18. `Code - Check Caption Timeline And QA Evidence` combines probe data with deterministic caption assertions.
19. `IF - Did Automated QA Pass?`. False sets `QA_FAILED`. A transport-corrupt render may retry once with the same render contract. Payload or visual failures route to a new asset version or `EDITOR_REQUIRED`.
20. `Switch - Route Render Purpose`. Review render creates/updates the Airtable review record and sets `NEEDS_REVIEW`. Final render returns its working Drive file ID to Workflow 6 without creating another review.
21. `Edit Fields - Return Render And QA Result`.

Binary stays in `$binary.data` until Drive upload. JSON transforms never sit between download and upload unless they explicitly preserve binary. A render failure never restarts transcription or analysis.

### Workflow 6: Review decision

**Input:** Airtable review update.  
**Output:** `{ok, review_id, asset_id, asset_version, decision, delivery_status}`.

Node sequence:

1. `Airtable Trigger - Watch Review Decisions` uses a Last Modified Time field.
2. `Code - Validate Review Decision And Feedback`.
3. `Crypto - Build Review Decision Key`.
4. `Airtable - Find Processed Review Decision`.
5. `IF - Is Review Decision New?`. False stops as a duplicate.
6. `Airtable - Load Current Asset Version`.
7. `IF - Does Review Match Current Asset Version?`. False marks the decision stale and stops.
8. `Switch - Route Review Decision` with `APPROVED`, `CHANGES_REQUESTED`, `REJECTED`, and fallback outputs.
9. Approved path: `Airtable - Bind Approval To Asset Version And Payload Hash`.
10. `Switch - Choose Approved Finalization`. Reuse the exact reviewed MP4 when production-ready. Otherwise call `Execute Sub-workflow - Render Approved Final Version` and require QA success with the unchanged payload hash.
11. `Google Drive - Search For Existing Final Delivery` by deterministic delivery key.
12. `IF - Is Final Delivery Already Present?`. False downloads the working MP4, uploads it to the final folder, creates the associated copy package, and uploads that package.
13. `Google Drive - Verify Final MP4` and `Google Drive - Verify Final Copy Package`.
14. `Airtable - Mark Asset Complete` stores file IDs, URLs, checksums, reviewer, and approval timestamp.
15. `Airtable - Recalculate Episode Completion`; then the approved path stops. No node follows final verification.
16. Changes-requested path requires written feedback and explicit `change_scope`, atomically increments `asset_version`, invalidates approval, and calls `Execute Sub-workflow - Regenerate Requested Asset Stage`.
17. Rejected path requires written feedback, marks only that asset `REJECTED`, recalculates the episode, and stops.
18. `Edit Fields - Return Review Decision Result`.

Human review occurs twice when needed: high context risk before rendering, then mandatory review of every completed asset. Silence leaves the record in `NEEDS_REVIEW`; it never approves. Face framing, caption readability, brand placement, pacing, and contextual fairness remain human judgments.

### Workflow 7: Error handling

**Typed input:** `workflow_name:string`, `execution_id:string`, `stage:string`, `episode_id:string`, `asset_id:string`, `asset_version:number`, `stage_run_key:string`, `attempt_number:number`, `error_code:string`, `error_message:string`, `retryable:boolean`. The same workflow also accepts Error Trigger payloads.  
**Output:** `{recorded, event_id, notification_status, terminal_state}`.

Node sequence:

1. `Error Trigger - Catch Unhandled Podcast Workflow Failure` and `Execute Workflow Trigger - Receive Structured Podcast Failure`.
2. `Edit Fields - Normalize Unhandled Failure` / `Edit Fields - Normalize Structured Failure`.
3. `Code - Redact Secrets And Classify Failure`.
4. `Crypto - Build Processing Event ID`.
5. `HTTP Request - Upsert Processing Event`.
6. `IF - Are Automated Attempts Exhausted?`. False records the retry schedule and returns. True continues.
7. `Switch - Choose Failed Record Type` updates the affected episode or asset to `FAILED` while preserving `last_completed_stage` and existing successful siblings.
8. `IF - Is Owner Notification Configured?`.
9. `HTTP Request - Notify Configured Workflow Owner` sends stage, record IDs, safe error text, attempt count, and resume point. It never includes tokens, credentials, transcript text, or provider response bodies.
10. `Airtable - Record Notification Result`.
11. `Edit Fields - Return Error Handling Result`.

If notification fails, the Processing Event remains the durable fallback with `notification_status=FAILED`. The notifier is not recursively connected to Workflow 7. Each production workflow must be assigned to this error workflow in n8n settings after build; that assignment requires separate verification.

## 6. Configuration placeholders and blockers

Coder must omit credential blocks until real credential IDs are selected. The following are configuration fields, not invented values: Airtable base/table IDs; Drive intake, working, review, and final folder IDs; AssemblyAI, OpenAI, Shotstack, Google Drive, and Airtable credentials; provider callback secrets; signed-media URL service; media-probe endpoint; alert endpoint; reviewer identity; candidate count; clip range; platforms; brand assets; font and colors; music; disclaimers; retention period; and episode budget.

Integrator must verify before Coder enables any related node:

1. Shotstack Rich Captions request shape, word-highlight behavior, prior-word reset, line grouping control, trim timing, stage watermark, callback/status fields, download expiry, idempotency, and current cost.
2. AssemblyAI upload or signed-URL route, diarization and word fields, callback signing, statuses, limits, idempotency, retention, and current cost.
3. OpenAI Responses structured-output request and response shape for the configured model, context limits, usage fields, idempotency behavior, and current price.
4. Airtable REST `performUpsert` or an equivalent atomic dedupe method, PAT scopes, rate limits, field limits, and Airtable Trigger Last Modified Time behavior.
5. Google Drive trigger metadata, source version/checksum fields, shared-drive support, binary upload/download, search by delivery key, returned checksums, and reviewer preview access.
6. A secure short-lived media URL route that AssemblyAI and the renderer can fetch. Private Drive preview links must not be treated as direct media URLs.
7. A media-probe service that can return duration, codecs, dimensions, audio presence, and playability. If unavailable, automated FR-10 proof is blocked.
8. The Remotion/FFmpeg fallback API contract and hosting only if Shotstack fails the spike.
9. The chosen notification channel and its authentication, limits, and failure behavior.

## 7. Acceptance evidence required from later stages

- Coder: inactive workflow IDs, retrieved node and connection maps, no credential values in JSON, paid nodes disabled, and zero-cost mocked executions.
- QA: exact input/output contracts, both outputs of every IF, fallback output of every Switch, both halves of each error output, idempotency checks before paid calls, and approval-version locking.
- Tester caption spike: one 20–30 second execution with render ID, timing manifest, payload hash, downloadable MP4, Drive file ID, checksum, and visual proof of all six caption rules.
- Tester full path: one 5–10 minute fixture, at most three candidates, one render, duplicate intake replay, callback replay, failed render resume, change request version increment, rejected sibling isolation, and approved final Drive verification.
- King: confirm no publication branch exists, final upload is the terminal action, cost exposure stayed within the approved test scope, and every unverified provider assumption is either proven or still blocked.

