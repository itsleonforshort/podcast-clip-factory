# Podcast Clip Factory

An [n8n](https://n8n.io) automation that turns a long podcast episode into short captioned
video clips, without a human touching an editor.

Drop an episode into a Google Drive folder. The system transcribes it, reads the transcript
to find the moments worth clipping, writes the copy for each one, renders them as captioned
video, and parks the results for a human to approve before anything is published.

**85 nodes in the combined pipeline. 348 across the seven modules.**

---

## How it works

```
Google Drive folder
        │
        ▼
1. Episode intake          register the episode, upload the audio
        │
        ▼
2. Transcription           AssemblyAI, polled until the transcript lands
        │
        ▼
3. Transcript analysis     GPT picks the moments worth clipping
        │
        ▼
4. Asset generation        GPT writes the copy and the caption text
        │
        ▼
5. Rendering and QA        Shotstack renders, then the output is probed
        │
        ▼
6. Review decision         a human approves or rejects in Airtable
        │
        ▼
   approved clips
```

`7. Error handling` sits beside all of it on an Error Trigger.

## What is in this repo

| File | Nodes | Triggered by | Talks to |
|---|---|---|---|
| [`workflows/podcast-clip-factory--full-pipeline.json`](workflows/podcast-clip-factory--full-pipeline.json) | 85 | Manual | AssemblyAI, OpenAI, Shotstack, Drive, Airtable |
| [`workflows/modules/1-episode-intake.json`](workflows/modules/1-episode-intake.json) | 35 | Google Drive | AssemblyAI, Drive, Airtable |
| [`workflows/modules/2-transcription-completion.json`](workflows/modules/2-transcription-completion.json) | 61 | Schedule + sub-workflow | AssemblyAI, Drive, Airtable |
| [`workflows/modules/3-transcript-analysis.json`](workflows/modules/3-transcript-analysis.json) | 56 | Sub-workflow | OpenAI, Drive, Airtable |
| [`workflows/modules/4-asset-generation.json`](workflows/modules/4-asset-generation.json) | 51 | Sub-workflow | OpenAI, Drive, Airtable |
| [`workflows/modules/5-video-rendering-and-qa.json`](workflows/modules/5-video-rendering-and-qa.json) | 63 | Sub-workflow | Shotstack, Drive, Airtable |
| [`workflows/modules/6-review-decision.json`](workflows/modules/6-review-decision.json) | 63 | Airtable + sub-workflow | Drive, Airtable |
| [`workflows/modules/7-error-handling.json`](workflows/modules/7-error-handling.json) | 19 | Error Trigger | Airtable |

The full pipeline is the seven modules flattened into one importable workflow, wired to a
Manual Trigger so it can be run end to end under supervision. Use the modules if you want
the system in production; use the full pipeline if you want to read it or test it in one go.

## Status — read this before you rely on it

**This is a working build, not a finished product.** The whole chain is wired, validated and
has run against live providers. It has not yet produced a finished, approved video end to
end. The last run reached the render stage and stopped on a source-media access error.

Treat it as a detailed, honest reference for how a pipeline like this is put together,
rather than something to point at a paying client on day one.

## What you need

Five credentials, connected inside n8n. None of them are in this repo.

| Service | n8n credential type | Used for |
|---|---|---|
| AssemblyAI | Header Auth | Transcription |
| OpenAI | OpenAI API | Choosing moments, writing copy |
| Shotstack | Header Auth | Rendering video with captions |
| Google Drive | OAuth2 | Source audio and finished files |
| Airtable | Personal Access Token | The job log and the review queue |

Shotstack is pointed at its **stage** environment throughout. Move it to production
deliberately, not by accident.

## Importing it

1. In n8n, open **Workflows**, then **Import from File**.
2. Pick the JSON you want from `workflows/`.
3. Open each node with a red credential warning and pick your own credential.
4. Replace every placeholder value (see the table below).
5. Leave the workflow **inactive** until you have run it once by hand and read the result.

### Placeholders you must replace

Every private value was replaced before publishing. Search for these and put your own in:

| Placeholder | Replace with |
|---|---|
| `appYOURBASEID0000` | Your Airtable base id |
| `tblYOURTABLE00001` … `00008` | Your Airtable table ids |
| `viwYOURVIEWID0000` | Your Airtable view id |
| `recYOURRECORDID01`, `recYOURRECORDID02` | Airtable record ids, only in the test build |
| `YOUR_INTAKE_FOLDER_ID` | The Drive folder you drop episodes into |
| `YOUR_WORKING_FOLDER_ID` | A Drive folder for work in progress |
| `YOUR_OUTPUT_FOLDER_ID` | The Drive folder finished clips land in |
| `YOUR_DRIVE_FILE_ID` | The Drive file id of your source audio |
| `YOUR_SUBWORKFLOW_ID_1`, `_2` | The n8n ids of the sub-workflows you import |
| `YOUR_AIRTABLE_CREDENTIAL`, `YOUR_OPENAI_CREDENTIAL`, and the other three | Your own n8n credential, picked in the UI |
| `you@example.com` | Your notification address |

`REDACTED_TEST_FIXTURE` marks three Code nodes in the full pipeline that held recorded
output from one specific test run, including transcript text from a real episode. The
recorded payloads were removed before publishing. Those nodes are recovery scaffolding for
that run, not part of the working pipeline — delete them, or feed them your own data.

## Things worth knowing before you change it

These cost real time to find.

- **Long jobs are asynchronous.** Transcription and rendering both submit, then poll on a
  Wait node. Nothing here assumes a render returns on the first response.
- **A timeout does not cancel a render.** It abandons it. The job keeps running and you are
  still billed for it, and a retry buys a second one.
- **A zero-valued optional property is not the same as an absent one.** Sending a zero for
  something you meant to leave out produces real, visible defects in the output.
- **Verify by measuring the finished file**, not by checking that nodes turned green. Every
  fault in this build was caught downstream of the node that caused it.
- **`$helpers` is undefined on some self-hosted instances.** Code nodes here use
  `this.helpers`.
- **`$json` does not exist in a Code node set to "Run Once for All Items".** Use
  `$input.first().json` or `$input.all()`.
- **`$('Node').first()` returns that node's last run in the whole execution**, not the run
  belonging to the current item. Per-item state is seeded once and passed along the item.
  `$runIndex` has the same flaw.

## Node naming

Every node is named `Original Node Type - What it actually does`, for example
`HTTP Request - Read AssemblyAI Transcript Status`. The first half keeps the node type
obvious on the canvas. The second half says why it is there. Both halves are required.

## What is not here

Deliberately left out: throwaway diagnostic workflows, caption-geometry spikes, test
fixtures, and the run-by-run evidence files from the build. They are scaffolding, not the
system.

## Design notes

Six write-ups from the build, in `docs/`. They are the reasoning behind the awkward parts,
not a tutorial.

| Doc | What it covers |
|---|---|
| [`01-system-architecture.md`](docs/01-system-architecture.md) | The whole design: the seven modules, the data model, retries, idempotency, where a human steps in |
| [`02-caption-timing-diagnostic.md`](docs/02-caption-timing-diagnostic.md) | How caption timing was measured when the captions drifted |
| [`03-caption-geometry-fix.md`](docs/03-caption-geometry-fix.md) | Why captions landed in the wrong place, and the fix |
| [`04-sentence-safe-clip-endings.md`](docs/04-sentence-safe-clip-endings.md) | Cutting a clip on a sentence boundary instead of mid-word |
| [`05-binary-preservation-fix.md`](docs/05-binary-preservation-fix.md) | Losing the binary between nodes, and keeping it |
| [`06-binary-to-drive-fix.md`](docs/06-binary-to-drive-fix.md) | Getting the rendered file into Drive intact |

The same placeholders apply in the docs as in the workflows.

## Licence

MIT. See [`LICENSE`](LICENSE).
