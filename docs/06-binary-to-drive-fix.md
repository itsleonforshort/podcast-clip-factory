# Podcast rendered-reel binary to Drive — fix design 526

**Architecture version:** 1.0
**Date:** 2026-09-12
**Workflow:** `YOUR_WORKFLOW_ID` — `Podcast Video Automation v1 - CONTROLLED RUNTIME TEST - INACTIVE`
**Answers:** `docs/reports/podcast-qa-binary-merge-525.md`, Blocking 1, plus rulings on N2, N3 and N5.
**Extends, does not replace:** `docs/architecture/podcast-binary-preservation-fix-525.md` v1.0.
**Author role:** Designer. Nothing here was built, deployed or executed. No n8n write tool was called.

---

## 0. One-paragraph summary

The Merge repair rescues the MP4 binary at `Code - Verify Downloaded MP4 Signature And
Preserve Binary`, and then `Code - Decide TEST Rendered Reel Write` throws it away by
returning a fresh JSON-only item. The fix is to re-attach the binary inside that same Code
node, by reference, from the Verify node. That is one node's code changed, no node added,
no edge changed. Two small side edits come with it: the Crypto node's implicit hash
settings get written out, and the evidence node stops hardcoding a check it never made.
Total cost of this change and its proof: **USD 0.00.**

---

## 1. The chosen approach, and why the other is rejected

### Chosen: repair the branch. Carry the binary forward to the upload.

### Rejected: pre-place the artifact in Drive so `needs_write` is false.

Four reasons, any one of which is enough.

1. **It can never satisfy QA's own acceptance criterion 10**, which requires
   `Google Drive - Save TEST Rendered Reel` to receive a binary and create a file whose
   bytes and SHA-256 match. Skipping the node cannot prove the node.
2. **It masks the repair rather than proving it.** Once the file exists with
   `artifact_key=rendered_reel_mp4`, the IF sends every future run down the false branch.
   The upload path would stay unexecuted forever, including on the first real podcast.
3. **It fabricates evidence.** The evidence JSON records `render.drive_file_id` and treats
   it as the workflow's own output. Pointing that at a hand-uploaded file makes the
   evidence a statement about something the workflow did not do. This project already has
   a written rule against checkers that report values they did not measure.
4. **It is a hand change to external state**, done once, outside version control, and not
   reproducible by anyone reading the workflow later.

The Orchestrator's stated view is correct and is adopted.

**Standing instruction that follows from this:** nobody may pre-place a file carrying
`artifact_key=rendered_reel_mp4` and `test_run_id=podcast-runtime-test-20260911-01` in
folder `YOUR_DRIVE_FILE_ID` before the repaired upload has run once. Doing
so would silently disarm the very test this fix exists to enable.

---

## 2. The mechanism — exactly how the binary reaches the upload

### 2.1 The route, unchanged in shape

```
Code - Verify Downloaded MP4 Signature And Preserve Binary   item has binary.data
  -> Google Drive - Find Existing TEST Rendered Reel          search; emits Drive rows, binary gone
  -> Code - Decide TEST Rendered Reel Write                   <<< THE ONLY EDIT
  -> IF - Does TEST Rendered Reel Need Creation?              passes items through untouched
  -> true  -> Google Drive - Save TEST Rendered Reel          reads binary field 'data'
  -> false -> Google Drive - Reconcile TEST Rendered Reel
```

No node is added. No connection is added, removed or re-indexed. The JSON shape leaving
`Code - Decide TEST Rendered Reel Write` is byte-identical to today, so the IF condition
`$json.needs_write===true` and everything downstream are untouched.

### 2.2 Why reading the binary back from the Verify node is sound — proven, not remembered

The mechanism depends on one fact: does `$('Node Name').first()` inside a Code node return
the whole item including its `binary` map, or only `json`?

Read from n8n's own source, `packages/workflow/src/workflow-data-proxy.ts`:

```ts
private returnExecutionData(data: INodeExecutionData | INodeExecutionData[], fullItem = false) {
	if (fullItem) return data;
	if (this.workflow.settings?.binaryMode !== BINARY_MODE_COMBINED) return data;

	if (Array.isArray(data)) {
		return data.map((i) => i.json);
	}

	return data.json;
}
```

The `binary` property is stripped **only** when `settings.binaryMode` equals the combined
mode. This workflow's `settings.binaryMode` is `separate`, which QA read live. So the full
item, binary included, is returned.

That is confirmed a second way, from a real run rather than from source. In combined mode
`.first()` returns the raw JSON object, so `.first().json` would be `undefined`. This
workflow uses `$('...').first().json` in dozens of expressions, and execution 525 ran
through 44 of them. If the mode were combined, the workflow would already be broken
everywhere. It is not. **`binaryMode` is `separate`, proven by a run.**

Under `separate` mode with filesystem binary storage, `binary.data` is a small descriptor
holding `id`, `fileName`, `mimeType`, `fileExtension` and `fileSize` — a pointer into the
execution's binary store, not 18 MB of bytes. Copying it into a new item costs nothing and
the Drive node resolves it when it calls `getBinaryDataBuffer`. This is the same object
the Verify node already hands on with `binary:item.binary`.

### 2.3 Does `Code - Verify Downloaded MP4 Signature And Preserve Binary` run exactly once?

**Yes. `.first()` is safe here.** Traced through the live connections:

- `IF - Does Shotstack Need Another Poll?` true output goes to
  `Wait - Pause Before Shotstack Status Check`, which is the poll loop.
- Its **false** output goes to `HTTP Request - Download TEST Rendered MP4`.

So the download sits on the loop's exit, not inside it. It runs once, Crypto runs once, the
Merge emits one item, Verify runs once. Last run and only run are the same run, so the
known trap — `.first()` returning a node's last run in the execution — cannot bite.

This is also not a new dependency. `Code - Validate Media Probe And Build TEST Evidence`
already reads `$('Code - Verify Downloaded MP4 Signature And Preserve Binary').first().json`
and has done since the workflow was built. The new code uses the identical reference.

**Recorded invariant:** this holds only while one render produces one file per execution.
The ratified 525 architecture already flags the same limit for the Merge node. If the
workflow ever renders more than one clip per run, both this reference and the Merge must be
re-designed to pair on a render identifier, not on position or on `.first()`.

### 2.4 The precise code

Replace the whole `jsCode` of `Code - Decide TEST Rendered Reel Write`. Keep
`mode: runOnceForAllItems` and `language: javaScript` exactly as they are.

```js
const rows=$input.all().filter(i=>i.json&&i.json.id);
if(rows.length>1) throw new Error('DRIVE_RENDERED_REEL_MP4_DUPLICATES_FOUND');
const needs_write=rows.length===0;
const verified=$('Code - Verify Downloaded MP4 Signature And Preserve Binary').first();
const bin=verified&&verified.binary?verified.binary:null;
if(needs_write&&!(bin&&bin.data)) throw new Error('RENDERED_REEL_MP4_BINARY_MISSING_BEFORE_UPLOAD');
const out={json:{needs_write,existing_file_id:rows[0]?.json?.id||'',artifact_key:'rendered_reel_mp4'}};
if(bin&&bin.data) out.binary={data:bin.data};
return [out];
```

Why each line is the way it is:

- `$input.all()` and never `$json`. The node is `runOnceForAllItems`, where `$json` does not
  exist on this instance. The existing first line already obeys this and is kept verbatim.
- No `$helpers`. Nothing on this instance has it. Nothing here needs it.
- **No `getBinaryDataBuffer` call.** The node must not load the file. It only needs to know
  the descriptor exists and to pass the pointer on. Loading 18.7 MB here would be pure waste
  and, worse, would read the *input* item — a Drive metadata row that has no binary at all.
- `out.binary={data:bin.data}` copies only the `data` key, the one
  `inputDataFieldName: "data"` reads. Copying one named key rather than the whole map keeps
  the contract explicit.
- The JSON object is unchanged in field names, order and values.
- The guard throws only when `needs_write` is true, because a missing binary is not a fault
  when nothing is going to be uploaded.

---

## 3. The IF node and the false branch

**Must the binary survive the IF?** Yes, on the true branch, or the upload has nothing to
read. The n8n IF node pushes whole items into its output branches without rewriting them,
so `binary.data` passes through untouched. QA confirmed this reading the node behaviour.
No change to the IF node is required or permitted.

**What should the false branch do with it?** Nothing. Leave it alone.

The false branch goes to `Google Drive - Reconcile TEST Rendered Reel`, a `fileFolder:search`
that ignores its input's binary and emits its own Drive rows. The binary is dropped there
naturally and correctly, because from that point on the run only needs the Drive file's
metadata.

**Explicitly rejected:** adding a node to strip the binary on the false branch. Under
`separate` binary mode the item carries a pointer, not bytes, so there is no memory to
reclaim. An extra node would add an edge, a name and a failure mode in exchange for nothing.

---

## 4. `Google Drive - Save TEST Rendered Reel` — what must be written out

The live node stores no `resource` and no `operation`, so it runs on n8n defaults. The
defaults happen to be right, but rule 2 of `CLAUDE.md` says a parameter that matters must be
written out, and this workflow's own local export already states them. This is drift between
live and export, not a redesign.

Write these three explicitly, matching the export exactly:

| Parameter | Value | Why it matters |
|---|---|---|
| `resource` | `"file"` | If the default ever changes, the node silently stops being an upload |
| `operation` | `"upload"` | Same. `upload` is what makes it a binary write at all |
| `inputDataFieldName` | `"data"` | Names the binary property. This is the exact field the fix in §2.4 populates. Leaving it implicit hides the whole contract |

`get_node` on `nodes-base.googleDrive` confirms `inputDataFieldName` is **required** under
`resource: file` + `operation: upload`, with default `data`.

**Everything else on this node stays exactly as it is.** `name`, `driveId`, `folderId`,
`options.fields`, `options.simplifyOutput`, `options.appPropertiesUi`, the credential,
`retryOnFail: false`, `maxTries: 1`, `onError: continueErrorOutput`. Do not touch them.

`retryOnFail: false` is deliberate and stays: an upload is a non-idempotent write, and a
retry could create a second Drive file that the duplicate check would then trip over.

The binary's `mimeType` and `fileName` ride along inside the copied descriptor, so the
uploaded file keeps `video/mp4`. The Drive file's visible name comes from the node's own
`name` expression, which is unchanged.

---

## 5. N5 — error routing, and the minimum change

**Ruling: accept N5, and fix it with the throw already written into §2.4. Change no wiring.**

QA is right that today a missing binary surfaces as
`DRIVE_RENDERED_REEL_MP4_RECONCILIATION_FAILED`, which names the wrong cause. The chain is:
upload throws, `onError: continueErrorOutput` sends it to the reconcile search, the search
finds zero rows, `Code - Reconcile TEST Rendered Reel` throws about reconciliation. A human
reading that message looks at Drive permissions, which is the wrong place.

The minimum change that fixes this is the guard in `Code - Decide TEST Rendered Reel Write`:

```js
if(needs_write&&!(bin&&bin.data)) throw new Error('RENDERED_REEL_MP4_BINARY_MISSING_BEFORE_UPLOAD');
```

It costs zero nodes and zero edges, it fires **before** the upload rather than after, and it
names the real cause. It is inside the one node already being edited.

**Explicitly rejected, and why:**

- **Splitting the Save node's success and error outputs onto different targets.** All five
  `Google Drive - Save ...` nodes in this workflow share one pattern: both outputs land on
  their reconcile node. Breaking that pattern on one of the five makes the canvas
  inconsistent and invites the same change on the other four. Leo objected to rebuilding
  when he did not ask for it, and this is rebuilding.
- **Adding a dedicated error-reporting node.** New node, new edges, no new information —
  the throw above already says everything the node would say.

The reconcile-on-both-outputs pattern stays. It fails closed, which is the important part.
QA logged it as a watch item and it remains one. **It should be revisited as a whole, across
all five Drive branches, when this workflow is prepared for production activation** — not
now, and not on one branch only.

---

## 6. N2 — implicit parameters on Merge and Crypto

**Ruling: accept the Crypto half. Reject the Merge half.**

### Crypto — accepted

`Crypto - Hash TEST Rendered MP4` stores only `binaryData` and `dataPropertyName` live, but
the local export already lists six parameters. Write the live node's `parameters` to match
the export exactly:

```json
{
  "action": "hash",
  "type": "SHA256",
  "binaryData": true,
  "dataPropertyName": "working_video_checksum",
  "encoding": "hex",
  "binaryPropertyName": "data"
}
```

`get_node` on `nodes-base.crypto` v2 confirms all six exist, that `type` defaults to
`SHA256` and `encoding` defaults to `hex`, and that both are marked required under
`action: hash`. So every written value equals the current effective value. **The checksum
cannot change.** QA's acceptance criterion 18 requires the execution-525 hash to stay
byte-identical, and it will.

QA's reason is the right one and it has teeth: if a future n8n version moves the hash `type`
default, every stored SHA-256 silently stops matching and nothing breaks loudly. This edit
also removes the live/export drift QA raised in N1.

### Merge — rejected, on evidence

The 525 architecture asked for `numberInputs: 2`, `mergeMode: "deepMerge"` and
`overrideEmpty: false`, and the Coder report claimed them. QA found none of the three
stored. I checked the schema:

- `numberInputs` **is real** on `nodes-base.merge` v3.2 under `mode: combine` +
  `combineBy: combineByPosition`, default `2`.
- `mergeMode` and `overrideEmpty` **do not exist anywhere in the 21 properties of Merge
  v3.2.** They are not under `options.clashHandling.values`, which holds only `resolveClash`.
  They belonged to an older Merge version.

So the 525 architecture asked for two parameters that are not part of this node. **That was
my predecessor document's error and it is corrected here.** Writing them in would put keys
into `parameters` that n8n ignores, and would tell every future reader that this node has
settings it does not have. That is worse than leaving them out.

That leaves only `numberInputs: 2`, which is cosmetic — the node has exactly two wired
inputs and the default is 2. Against that: the Orchestrator's instruction freezes
`Merge - Restore TEST Rendered MP4 Binary After Hash` and its four edges, QA proved the node
correct, and `resolveClash: preferLast` is the one parameter that mattered and it is
correctly stored. **A frozen, proven node is not worth reopening for a cosmetic default.**

**Deferred, with a named trigger:** write `numberInputs: 2` out the next time this Merge node
is edited for any other reason. Do not open it for this alone.

### The report wording, N1

Separately from the node itself, the Coder's parity claim must be corrected in the report:
live and export are **semantically equivalent**, not identical, and the differences are the
8px grid position snap plus the parameters n8n does not store because they equal defaults.
That is a documentation fix, not a workflow fix. QA's criterion 17 asks for exactly this.

---

## 7. N3 — the check that was never measured

**Ruling: accept. Fix it properly, by moving the measurement to the node that makes it.**

`Code - Validate Media Probe And Build TEST Evidence` writes `ftyp_signature_valid: true` as
a literal inside its `checks` object. That node never looks at the file's bytes — its input
is the Shotstack probe response, and the file itself is long gone from the item. So the
saved evidence JSON reports a measurement that node did not take.

Renaming the field alone would be honest but still hardcoded, and the value would stay `true`
no matter what happened upstream. The better fix is the same size and is genuinely derived.

**Two paired edits. They must land in the same change or the workflow fails closed.**

**Edit A — `Code - Verify Downloaded MP4 Signature And Preserve Binary`.** This node does
measure the signature. Have it record what it found. Change only the returned JSON, adding
one field after the spread:

```js
return [{json:{...p,working_video_checksum:item.json.working_video_checksum,downloaded_bytes:b.length,ftyp_signature_valid:true},binary:item.binary}];
```

Everything before that line — the `getBinaryDataBuffer` call, the `RENDER_BINARY_MISSING`
throw, the `ftyp` test, the `RENDER_MP4_BINARY_CONTRACT_INVALID` throw, `binary:item.binary`
— is unchanged. The literal `true` is honest **here**, because this line is only reached
when the check passed two lines earlier.

**Edit B — `Code - Validate Media Probe And Build TEST Evidence`.** Inside the `checks`
object, change one entry:

```js
ftyp_signature_valid:true
```

to

```js
ftyp_signature_valid:b.ftyp_signature_valid===true
```

`b` in that node is already `$('Code - Verify Downloaded MP4 Signature And Preserve Binary').first().json`.
Nothing else in the node changes.

**Why paired:** `deterministic_pass` requires every value in `checks` to be truthy. If Edit B
ships without Edit A, the field reads `undefined`, the check goes false and the run throws
`TEST_MEDIA_PROBE_QA_FAILED`. That direction is safe — it fails closed rather than passing a
lie — but it would waste a paid render. **Coder must apply A and B together, and QA must
verify both before any run.**

This satisfies QA's acceptance criterion 20.

---

## 8. The complete change list for the Coder

Four nodes. No node added. No node deleted. No connection touched. Node count stays 85,
connection count stays 133.

| # | Node | Change | Risk |
|---|---|---|---|
| 1 | `Code - Decide TEST Rendered Reel Write` | Replace `jsCode` with §2.4 | The fix. JSON output shape unchanged |
| 2 | `Google Drive - Save TEST Rendered Reel` | Write out `resource`, `operation`, `inputDataFieldName` (§4) | None. Values equal current defaults |
| 3 | `Crypto - Hash TEST Rendered MP4` | Write `parameters` out in full (§6) | None. Values equal current effective config |
| 4a | `Code - Verify Downloaded MP4 Signature And Preserve Binary` | Add `ftyp_signature_valid:true` to the returned JSON (§7 Edit A) | None. Additive field |
| 4b | `Code - Validate Media Probe And Build TEST Evidence` | One `checks` entry becomes derived (§7 Edit B) | Must ship with 4a |

### Hands off. These belong to other work in flight.

- `Code - Parse Validate Moment Copy And Build VTT Render Contract` — a Coder is editing it
  now for the sentence-safe ending.
- `Merge - Restore TEST Rendered MP4 Binary After Hash` and its four edges — proven correct,
  frozen.
- `render_key` and `RETIRED_RENDER_KEYS` — the RENDER-03 bump is a separate edit that needs
  Leo's spend approval first.
- Caption highlight timing and cue durations — the separate A/B diagnostic owns these.
- The other four `Google Drive - Save ...` branches and their reconcile wiring — see §5.

Use `n8n_update_partial_workflow`. Send each node's `parameters` object complete, because a
partial parameters object on a node like Crypto would drop the keys not sent.

---

## 9. The decisions this architecture is required to state

**Node sequence.** Unchanged from the live workflow. The five-node stretch in §2.1 is the
whole scope. Real node types: `n8n-nodes-base.code` v2, `n8n-nodes-base.googleDrive` v3,
`n8n-nodes-base.if` v2.3, `n8n-nodes-base.crypto` v2.

**Node names.** Every name already follows `Original Node Name - Action`. No node is renamed,
so the convention is preserved by doing nothing.

**Sub-workflows.** Does not apply. This is one Code node's body plus three parameter
write-outs inside a single flow. Extracting it would push a binary reference across a
workflow boundary, which adds a real failure mode in exchange for no reuse.

**Retry strategy.** Unchanged, and deliberately so.
`HTTP Request - Download TEST Rendered MP4` keeps 3 tries with 5 s fixed backoff — it is an
idempotent GET. The two Drive search nodes keep 3 tries with 3 s backoff — idempotent reads.
`Google Drive - Save TEST Rendered Reel` keeps `retryOnFail: false, maxTries: 1`, because a
retried upload could create a second file. Code nodes get no retries; a deterministic
function that failed once fails again.

**Idempotency.** Unchanged. The dedupe key for this artifact is the pair of Drive
appProperties `test_run_id` + `artifact_key`, searched inside one folder, evaluated by
`Code - Decide TEST Rendered Reel Write` into `needs_write`. For this run that is
`test_run_id=podcast-runtime-test-20260911-01` and `artifact_key=rendered_reel_mp4` in folder
`YOUR_DRIVE_FILE_ID`. A second run finds the file, sets `needs_write` false,
skips the upload and reconciles. The fix does not alter this and must not.

**Database usage.** No new storage. No Airtable field added, no Drive artifact added, no
workflow static data. The only new in-flight value is the binary pointer already created by
the download, carried one extra hop, plus one boolean on the Verify node's JSON.

**Human-review points.** None added. This workflow is inactive, manually triggered and run
by Leo clicking once. The existing human gate is unchanged: caption timing stays
`PENDING_HUMAN_REVIEW` in the evidence JSON, and no publishing or activation is in scope.

**Error routing.** One new precise failure, `RENDERED_REEL_MP4_BINARY_MISSING_BEFORE_UPLOAD`,
thrown before the upload. All existing routes unchanged: Download error output to
`Stop and Error - Fail Closed At External Boundary`, both Drive search error outputs to the
same Stop and Error, both Save outputs to the reconcile search. Every path still fails
closed. No Error Trigger is added, because this workflow is never unattended — it does not
run without a human click, and n8n's saved error execution is the notification surface. An
Error Trigger becomes required at production activation, which is out of scope here.

**Cost exposure.** **USD 0.00 for every part of this change and its proof.** No node in the
change list calls a paid API. The Code nodes are local. The Crypto node is local. The Drive
upload is free storage. The zero-cost proof in §10 makes no paid call either. The spent
`RENDER-02` key stays spent and stays untouched.

---

## 10. How to prove this without paying — the honest answer

### The constraint, stated plainly

QA's Blocking 2 is correct. `render_key` is the literal `...|RENDER-02`, that key is already
recorded in Airtable against a real job id, and
`Code - Refuse Duplicate Shotstack Submission And Build Claim` therefore throws
`DUPLICATE_SHOTSTACK_PAID_SUBMISSION_BLOCKED` long before the download branch. There is no
second path into `HTTP Request - Download TEST Rendered MP4`. **Clicking Execute on
`YOUR_WORKFLOW_ID` right now proves nothing about this fix and is not worth doing.** It is
also free, because it stops before every paid node.

### What cannot be used

**Pinned data cannot prove this.** n8n's pin data holds JSON only; binary is not preserved in
pinned data. Pinning the Verify node would produce an item with no `binary`, so the fix would
report its own new error and the test would prove the opposite of what is wanted. `pinData`
is `{}` today and it should stay that way. Rule this route out and do not revisit it.

**Reading the live JSON is not proof either.** It shows the edit landed. It does not show
that a Code node's returned binary reference survives a Drive search boundary and an IF into
an upload. On this instance, "it exists in the catalogue" has already been proven not to mean
"it works here".

### What can be used — a throwaway probe workflow, one click, USD 0.00

Coder builds a small separate workflow. This is the pattern already accepted on this project
for the VTT A/B question, so it is not a rebuild of anything.

**Name:** `Podcast Binary Route Probe - THROWAWAY - INACTIVE`

**Shape — it reproduces the exact stretch under test and nothing else:**

1. `Manual Trigger - Start Binary Route Probe`
2. `Code - Build Fake MP4 Binary For Probe` — creates about 32 bytes whose 5th to 8th bytes
   spell `ftyp`, so the real Verify code accepts it. No network, no cost, fully deterministic:

   ```js
   const head=Buffer.from([0x00,0x00,0x00,0x20,0x66,0x74,0x79,0x70,0x69,0x73,0x6f,0x6d]);
   const buf=Buffer.concat([head,Buffer.alloc(20,0)]);
   const bd=await this.helpers.prepareBinaryData(buf,'binary-route-probe.mp4','video/mp4');
   return [{json:{probe:true},binary:{data:bd}}];
   ```

   **Fallback if `this.helpers.prepareBinaryData` turns out not to be exposed in the Code
   node on this instance:** replace this node with
   `HTTP Request - Download Tiny Public MP4`, `responseFormat: file`,
   `outputPropertyName: data`, against a small public MP4 the Integrator has verified is
   anonymous, free and under 2 MB. Coder reports which route was used. The origin of the
   bytes is irrelevant to what is being tested.
3. `Crypto - Hash Probe Binary` — the same six parameters as §6.
4. `Merge - Restore Probe Binary After Hash` — the same parameters and the same four edges
   as the frozen production Merge. Wired the same way: step 2 fans to Crypto and to Merge
   input 0; Crypto to Merge input 1.
5. `Code - Verify Downloaded MP4 Signature And Preserve Binary` — **named exactly that**, and
   holding the production code string character for character, including §7 Edit A. Same name
   means the reference string inside the next node is identical, so the probe tests the real
   code and not a paraphrase of it.
6. `Google Drive - Find Existing Probe Artifact` — search on
   `artifact_key='binary_route_probe'`, `alwaysOutputData: true`.
7. `Code - Decide TEST Rendered Reel Write` — **named exactly that**, holding the §2.4 code
   character for character.
8. `IF - Does Probe Need Creation?` — the same boolean test on `needs_write`.
9. `Google Drive - Save Probe Artifact` — `resource: file`, `operation: upload`,
   `inputDataFieldName: "data"`, name `binary-route-probe.mp4`, appProperties
   `test_run_id=binary-route-probe-20260912` and `artifact_key=binary_route_probe`.

**Contamination guard.** The production searches filter on
`artifact_key='rendered_reel_mp4'` and on `test_run_id=podcast-runtime-test-20260911-01`.
The probe uses neither value, so its file cannot be seen by the real workflow's idempotency
check. Put it in a scratch folder if one is easy; the shared test folder is acceptable given
the guard. Delete the probe file and the probe workflow once the answer is recorded.

**What one click proves, at USD 0.00:**

- A Code node can return a binary reference read from another node via `.first()`.
- That reference survives the Drive-search boundary and the IF node.
- `Google Drive - Save ...` with `inputDataFieldName: "data"` accepts it and creates a real
  file.
- The `RENDERED_REEL_MP4_BINARY_MISSING_BEFORE_UPLOAD` guard does **not** fire when the
  binary is present. Running the probe a second time additionally proves the false branch
  behaves, since `needs_write` becomes false.

**What it does not prove, stated plainly:** it does not prove the production workflow's own
nodes and wiring executed. Node names and code strings are identical, but the run is a
different run. **Full proof of the real branch — QA's acceptance criteria 8, 9, 10 and 11 —
can only come from the next authorised paid render under a fresh `RENDER-03` key.** I am not
going to invent a test that pretends otherwise.

**Before any click on the real workflow**, the Tester must still do QA's criterion 13: read
the Airtable row for `event_id = TEST-20260911-01|SHOTSTACK` and confirm `payload_hash` is
`podcast-runtime-test-20260911-01|RENDER-02` with a non-empty `provider_job_id`. If
`payload_hash` is anything else, the code falls into the `superseded_prior_render` branch and
submits a **new paid render**.

### Sequence

1. Coder applies §8 items 1 to 4b to `YOUR_WORKFLOW_ID`.
2. Coder builds the probe workflow from §10. Inactive, `availableInMCP` false.
3. QA re-reads both. It checks the four production edits, that no edge or node count moved,
   and that the probe's two shared node names and code strings are character-identical.
4. Leo opens the probe workflow and clicks **Execute workflow exactly once**. Free.
5. Tester records whether a Drive file appeared and whether its byte size matches the probe
   buffer. Tester deletes the probe file afterwards.
6. Production proof waits for RENDER-03, which needs Leo's spend approval and a separate
   edit that retires RENDER-02.

---

## 11. Open questions Leo must answer

**One question, and it is the only one.**

The repaired upload path cannot be proven on the real workflow without a new paid Shotstack
render under a fresh `RENDER-03` key. The free probe in §10 proves the mechanism but not the
real branch.

**Is the free probe enough for now, or should the RENDER-03 spend be prepared straight
away?** The probe costs nothing and answers the mechanism today. The real proof costs one
Shotstack render, which the workflow's own model prices at USD 0.225 against the approved
USD 0.50 ceiling.

Everything else in this document is settled and needs no decision.

---

## 12. What the Integrator must verify

Three items, all free, none urgent enough to block the Coder on items 1 and 3 of §8.

1. **Does `this.helpers.prepareBinaryData` exist in the Code node on this instance?** This
   instance strips things the catalogue advertises. If it is missing, the probe uses the
   HTTP fallback and the Integrator supplies the URL in item 2. `getBinaryDataBuffer` is
   already proven here; `prepareBinaryData` is not.
2. **A small public MP4 URL**, only if item 1 comes back missing. Anonymous, free, under
   2 MB, real `ftyp` box, stable. Do not invent one from memory.
3. **Does the Shotstack `stage` environment bill?** QA raised this and it is still open. The
   endpoint is `https://api.shotstack.io/edit/stage/render` and the workflow's cost model
   charges USD 0.30 per minute regardless. It changes nothing in this document — every
   change here is free — but it decides what the USD 0.50 ceiling actually means when
   RENDER-03 is discussed.

No provider request body, endpoint, header or auth changes in this fix. Google Drive,
Airtable, AssemblyAI, OpenAI and Shotstack are all called exactly as before.

---

## 13. Acceptance — what QA should check when this comes back

These extend QA's existing twenty criteria; they do not replace them.

1. `Code - Decide TEST Rendered Reel Write` holds the §2.4 code character for character, is
   still `runOnceForAllItems`, and its returned JSON still has exactly the three fields
   `needs_write`, `existing_file_id`, `artifact_key` with the same values as before.
2. That node contains no `$json`, no `$helpers`, and no `getBinaryDataBuffer` call.
3. `Google Drive - Save TEST Rendered Reel` stores `resource: "file"`,
   `operation: "upload"` and `inputDataFieldName: "data"` explicitly, and every other
   parameter, its credential, `retryOnFail: false` and `maxTries: 1` are byte-identical to
   before.
4. `Crypto - Hash TEST Rendered MP4` stores all six parameters from §6, and `type` is
   `SHA256` with `encoding` `hex`.
5. `Code - Verify Downloaded MP4 Signature And Preserve Binary` returns
   `ftyp_signature_valid: true`, and everything above that return line is unchanged.
6. `Code - Validate Media Probe And Build TEST Evidence` derives `ftyp_signature_valid` from
   the Verify node and contains no hardcoded `true` for it.
7. Node count is still 85. Connection count is still 133. The `connections` object is
   byte-identical to the pre-change version.
8. `Merge - Restore TEST Rendered MP4 Binary After Hash` is untouched, including its four
   edges and `resolveClash: preferLast`.
9. `Code - Parse Validate Moment Copy And Build VTT Render Contract`, `render_key` and
   `RETIRED_RENDER_KEYS` are untouched by this change.
10. `n8n_validate_workflow` returns `valid=true`, 0 errors, 0 warnings.
11. `active=false`, `activeVersionId=null`, `settings.availableInMCP=false`, `pinData` still
    `{}`.
12. The probe workflow exists, is inactive, is MCP-off, and its two shared node names and
    code strings match production character for character.
13. No paid node ran. No new Shotstack, AssemblyAI or OpenAI job id exists.
