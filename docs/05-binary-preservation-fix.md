# Podcast binary preservation fix after execution 525

**Architecture version:** 1.0  
**Date:** 2026-09-12  
**Workflow:** `YOUR_WORKFLOW_ID` — `Podcast Video Automation v1 - CONTROLLED RUNTIME TEST - INACTIVE`  
**Decision:** **RATIFIED.** Add one Merge node after Crypto. This is the smallest safe repair for the proven `RENDER_BINARY_MISSING` failure.

## Decision basis

Execution 525 proves that `HTTP Request - Download TEST Rendered MP4` emitted one item with `$binary.data`, while `Crypto - Hash TEST Rendered MP4` emitted the expected SHA-256 value and removed the binary. This matches the current Crypto v2 implementation. The Merge v3.2 contract supports combining two inputs by item position and merging their binary maps.

The n8n template catalogue was searched for workflows using Merge and Crypto. Results showed ordinary Merge patterns, including template 5338 by Dr. Firas, but no template demonstrated binary restoration after Crypto. No template was adopted, so no template attribution is carried into the build.

Current n8n-MCP schemas were read for every node in this bounded flow: HTTP Request, Crypto, Merge, Code, and Stop and Error.

## Exact node sequence

1. **`HTTP Request - Download TEST Rendered MP4`**  
   Type `n8n-nodes-base.httpRequest`, version 4.5. Keep its current configuration. Its success output must emit exactly one item with binary property `data`. Keep `retryOnFail: true`, `maxTries: 3`, and `waitBetweenTries: 5000`. Keep `onError: continueErrorOutput`.

2. **`Crypto - Hash TEST Rendered MP4`**  
   Type `n8n-nodes-base.crypto`, version 2. Keep its current configuration: `action: hash`, `type: SHA256`, `binaryData: true`, `binaryPropertyName: data`, `dataPropertyName: working_video_checksum`, and `encoding: hex`. No retries apply because hashing is a deterministic local operation.

3. **`Merge - Restore TEST Rendered MP4 Binary After Hash`**  
   Type `n8n-nodes-base.merge`, version 3.2. Add this one node with the exact parameters below:

   ```json
   {
     "mode": "combine",
     "combineBy": "combineByPosition",
     "numberInputs": 2,
     "options": {
       "clashHandling": {
         "values": {
           "resolveClash": "preferLast",
           "mergeMode": "deepMerge",
           "overrideEmpty": false
         }
       }
     }
   }
   ```

   Input 1 is the untouched Download output. Input 2 is the Crypto output. `preferLast` means the Crypto JSON wins on clashes while the Merge output also receives the binary map from Input 1.

4. **`Code - Verify Downloaded MP4 Signature And Preserve Binary`**  
   Type `n8n-nodes-base.code`, version 2. Keep its current code unchanged. It must receive one item with `$binary.data` and `working_video_checksum`. It remains responsible for reading the bytes, checking the MP4 `ftyp` signature, requiring the checksum, setting `downloaded_bytes`, and returning the binary unchanged.

5. **`Google Drive - Find Existing TEST Rendered Reel`**  
   Existing downstream node. Its connection from Verify stays unchanged.

## Exact connections

- Keep `HTTP Request - Download TEST Rendered MP4` `main[0]` -> `Crypto - Hash TEST Rendered MP4` input index `0`.
- Add `HTTP Request - Download TEST Rendered MP4` `main[0]` -> `Merge - Restore TEST Rendered MP4 Binary After Hash` input index `0`.
- Add `Crypto - Hash TEST Rendered MP4` `main[0]` -> `Merge - Restore TEST Rendered MP4 Binary After Hash` input index `1`.
- Add `Merge - Restore TEST Rendered MP4 Binary After Hash` `main[0]` -> `Code - Verify Downloaded MP4 Signature And Preserve Binary` input index `0`.
- Remove only `Crypto - Hash TEST Rendered MP4` `main[0]` -> `Code - Verify Downloaded MP4 Signature And Preserve Binary` input index `0`.
- Keep `HTTP Request - Download TEST Rendered MP4` error output `main[1]` -> `Stop and Error - Fail Closed At External Boundary` input index `0`.
- Keep `Code - Verify Downloaded MP4 Signature And Preserve Binary` `main[0]` -> `Google Drive - Find Existing TEST Rendered Reel` input index `0`.

## Cardinality and correctness invariant

This design is approved only for the current one-render path. The completed Shotstack poll emits one render item. The Download node therefore emits one file item, and Crypto emits one checksum item. Combine by position is deterministic under that one-item invariant.

If the workflow later supports more than one rendered file in a run, this design must be reviewed. Pairing several items by position could attach a checksum to the wrong file. A future multi-render version must combine on a stable render identifier instead.

## Failure routes

- Download transport failures retain the existing three attempts with five-second fixed backoff. Exhaustion routes through the Download error output to `Stop and Error - Fail Closed At External Boundary`.
- Crypto errors use n8n's default fail-stop behavior. No downstream write runs.
- Merge configuration or runtime errors use n8n's default fail-stop behavior. No downstream write runs.
- A missing binary at Verify throws `RENDER_BINARY_MISSING`.
- A missing checksum or invalid MP4 signature at Verify throws `RENDER_MP4_BINARY_CONTRACT_INVALID`.
- Any of those local failures remain visible in the saved n8n error execution. They must never continue to the Google Drive write path.

No additional notification node applies. This is an inactive, manually run controlled-test workflow, so the n8n execution error is the current notification surface. A production activation design would need a separate notification route, but activation is outside this fix.

## Idempotency, stored data, and keys

This patch creates no external call and no persistent write. It introduces no new dedupe key. Existing upstream paid-render protection and the existing `test_run_id | render_key` ownership remain unchanged.

No database, Airtable row, Drive artifact, or workflow static data is added. The only added in-flight fields are the restored `$binary.data` map and the already generated JSON value `working_video_checksum`. Existing render context stays on the Crypto JSON and wins clashes through Input 2.

## Sub-workflow, review, and cost boundaries

A sub-workflow does not apply. The repair is one local join inside a single flow, and moving binary across another workflow boundary would add risk without reuse.

QA must verify the node parameters, connection indexes, inactive state, `availableInMCP: false`, and live/export parity before any execution. Tester should first replay the preserved execution-525 artifact through Download-equivalent input, Crypto, Merge, and Verify. A new Shotstack render is unnecessary for this repair.

Paid-call exposure is **USD 0** for the change and replay test. The Merge node is local. The existing `RENDER-02` key remains spent. Any later paid main render requires a new key, retirement of `podcast-runtime-test-20260911-01|RENDER-02` in the same edit, QA approval, and a fresh user approval for spend.

## Blocking questions and Integrator handoff

No blocking question remains for Coder. Integrator has verified the relevant n8n Crypto and Merge contracts, and execution 525 provides the runtime evidence.

No external API requires new verification because this fix does not change Shotstack, Google Drive, Airtable, AssemblyAI, or OpenAI requests. The existing rendered-file URL and download contract remain unchanged. Integrator must re-open the design only if Coder proposes a second download, a provider request change, or a multi-item render path.

