# File checkpointing & rewind

This file describes the **steps followed** to add Concept 17 (**File checkpointing & rewind**) to the Claude Agent
SDK Lab: what was read, what was decided, how it was tested, and what the tests changed.
To learn the concept itself, read [Tab17-Checkpointing-and-rewind.md](Tab17-Checkpointing-and-rewind.md).

| Concept | Topic | Routes | Explanation |
|---|---|---|---|
| 17 | `enableFileCheckpointing`, `Query.rewindFiles()` (`dryRun`), `RewindFilesResult`, user message `uuid`, `extraArgs: { "replay-user-messages": null }`, rewind after `resume` | `/api/c17/session`, `/send`, `/rewind`, `/end`, `/oneshot`, `/resume-rewind` | [Tab17-Checkpointing-and-rewind.md](Tab17-Checkpointing-and-rewind.md) |

## How to run it

```powershell
npm run dev        # server on http://localhost:3001, web on the Vite port
```

`node_modules` was copied from sample16, so `npm install` is not needed. Open the **17. Checkpointing & rewind** tab.
Start it from a normal terminal: from inside Claude Code, the inherited `CLAUDE_CODE_*` variables make every run use
the login (`apiKeySource: "none"`, see Tab16).

> **Only one sample can run at a time.** Every sample's server uses port **3001**. Stop the other samples'
> `npm run dev` first.

## Step 1: Choose the feature

The request said "implement the following feature sample" with no feature text. `sample17/` was a copy of sample16
(without `node_modules`). Four topics were offered: **File checkpointing & rewind**, **Sandbox**, **Plugins** and
**Query control methods**. **File checkpointing & rewind** was chosen.

## Step 2: Read the existing samples

| Read | To learn |
|---|---|
| `server/concepts/12-streaming-input.ts`, `Concept12StreamingInput.tsx` | The push queue, the `control()` route wrapper, the timeline, why a string prompt closes the `Query` |
| `server/concepts/16-settings-env.ts`, `Concept16SettingsEnv.tsx` | The `BASE` options pattern, run cards, "the browser never sends a path" |
| `server/concepts/06-sessions.ts` | `resume` |
| `server/index.ts`, `server/sse.ts`, `src/lib/sse.ts`, `src/App.tsx`, `src/styles.css`, `Tab16-Settings-and-env.md` | Mounting a router, SSE, the tab list, CSS to reuse, the doc style |

## Step 3: Check the types

`sdk.d.ts` (`0.3.281`): `Options.enableFileCheckpointing`, `Query.rewindFiles(userMessageId, { dryRun })`,
`RewindFilesResult` (`canRewind`, `error`, `filesChanged`, `insertions`, `deletions`, `skippedLinks`),
`SDKUserMessage.uuid` (optional, yours), `SDKUserMessageReplay` (`uuid`, `isReplay: true`), `Query.close()`.

## Step 4: Experiment before designing

Three scratch scripts called the SDK directly, with the `CLAUDE*` variables removed:

1. **Streaming input, three turns** (`Write`, then two `Edit`s, then Bash + `Write`), each message with its own
   `uuid`; then dry-run and real rewinds to every message, forward and back, and an unknown uuid.
2. **String prompt**: with and without `replay-user-messages`, `rewindFiles()` after the loop, then a resumed
   `query()` that rewinds; checkpointing off.
3. **A live session, on and off**: does the echo carry our uuid, what does rewind say when off, and does the model
   remember the rewound edit.

They showed that your `uuid` is the checkpoint id, that rewinding to message N restores the state *before* N, that
later files are deleted, that forward rewinds work, that a real rewind does not list files, that Bash changes are not
tracked, that off means `canRewind: false` for a dry run but a throw for a real one, that a finished string-prompt
`Query` throws `Query closed before response received`, and that a resumed query can rewind. The full table is in
Step 2 of the Tab.

The first run of script 1 changed nothing: its folder was under the 8.3 short path `C:\Users\LUIS~1.COC\…`, and no
file appeared there. Moving the folder under the project fixed it. It was not investigated further.

## Step 5: Design the concept

- **One fixed agent.** Haiku, `tools: Read/Write/Edit/Bash`, `settingSources: []`, `replay-user-messages` on, in a
  folder the server empties and seeds (`plan.md`, `config.json`) on every run.
- **Part A as a live session** (the Concept 12 queue): the server sets each message's `uuid` and sends it as a
  `checkpoint` event, so every message card gets *Preview* and *Rewind* buttons at once. A `files` event after every
  result and every real rewind shows the disk, not the model's opinion of it.
- **Presets that make the three lessons visible**: a tracked `Edit`, a tracked `Write` + `Edit`, and Bash changes
  (a new file and a `sed -i` on a tracked file); then *Ask from memory* and *Read the files again*.
- **Part B as two steps**: `/oneshot` (string prompt, shows the uuid echo and the error after the loop) and
  `/resume-rewind` (a new `query({ resume })` whose input never sends a message, then `rewindFiles()` and `close()`).
- **The browser never sends a path**: rewinds take a uuid, checked against a uuid pattern, and `sessionId` too.

## Step 6: Implement it

| File | What was done |
|---|---|
| `server/concepts/17-checkpointing.ts` | New: `/session`, `/send`, `/rewind`, `/end`, `/oneshot`, `/resume-rewind` |
| `server/index.ts` | Mounted on `/api/c17` |
| `src/concepts/Concept17Checkpointing.tsx` | New: `Files`, `RewindLine`, `Timeline`, `LivePart` (A), `AfterPart` (B) |
| `src/App.tsx` | The tab |
| `.gitignore` | `checkpoint-lab/` |

`npx tsc --noEmit -p .` passed, and `npx vite build` succeeded.

## Step 7: Test the routes

Only the Concept 17 router was mounted in a scratch server on port **3017**, started with `.env` loaded and the
`CLAUDE*` variables removed, and driven by a Node script like the browser does:

| Test | Result |
|---|---|
| Part A, three presets | The echoed uuid matches the one `/send` returned, each time |
| Part A, dry → #1, dry → #2 | 3 files each; +2 −4 and +2 −3; 10 to 30 ms |
| Part A, real → #2, → #3 (forward), → #1 | Files as expected; `bash-note.txt` survives all three; the `sed` change on `plan.md` is overwritten |
| Part A, `"../etc"` as uuid / an unknown uuid | 409 `Not a uuid.` / `canRewind: false`, `No file checkpoint found for this message.` |
| Part A, *Ask from memory* after rewinding to #1 | Answered `1.0.0` (the rewound value). In script 3 it had described the old content |
| Part A, checkpointing off | Dry runs: `canRewind: false`, `File rewinding is not enabled.`; real rewinds: 409 with the same text |
| Part A, rewind after `/end` | 409 `No open session with that id` |
| Part B, `/oneshot` | Echo with `uuid` and `session_id`; after the loop: `Query closed before response received` |
| Part B, `/resume-rewind` dry, then real | 1.8 s / 2.2 s; `config.json`, `release.md` listed, then restored and deleted |
| Part B, bad / unknown `sessionId` | 400 / 409 `No conversation found with session ID: …` |

One change came from the tests:

1. With `permissionMode: "dontAsk"` and `allowedTools: ["Bash(echo:*)"]`, the command `echo 'written by bash' >
   bash-note.txt` was denied, and the model created the file with `Write` instead, so it was tracked and the lesson
   disappeared. The agent now uses `permissionMode: "acceptEdits"` with only `Read` pre-approved: edits and
   filesystem commands inside `cwd` are accepted, anything else is denied (there is no `canUseTool`). Both Bash
   commands then ran, and the rewinds showed what Bash changes do.

Costs: $0.006 to $0.024 per turn on Haiku, less than $0.40 for all the tests. Nothing for the rewinds.

## Step 8: Run it in the real app

The real server (`server/index.ts`) was started on port **3001** and Vite on port **5199**: it served the page,
`App.tsx` with the new tab, and `Concept17Checkpointing.tsx` (HTTP 200). The Part B test was repeated through the Vite
proxy with the same results (`/resume-rewind` 2.0 s / 2.3 s). Both processes were stopped afterwards.

## Files added or changed

| File | Change |
|---|---|
| `server/concepts/17-checkpointing.ts` | New: the Concept 17 routes |
| `server/index.ts` | Mounts `/api/c17` |
| `src/concepts/Concept17Checkpointing.tsx` | New: the Checkpointing & rewind tab |
| `src/App.tsx` | Tab |
| `.gitignore` | Ignores `checkpoint-lab/` |
| `Tab1-query().md` | Adds Concept 17 to the table |
| `Tab17-Checkpointing-and-rewind.md` | Explanation of the concept |
| `Build-steps.md` | This file |
| `readme.md` | Same content as `Tab17-Checkpointing-and-rewind.md` |
