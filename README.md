# oar-coxswain

An intentionally small Electron cockpit for dogfooding
[`@botiverse/oar`](https://github.com/botiverse/oar).
Each window owns an `AgentHost` that can run one or more independent session
lanes. Humans inject through `prompt` or `steerOrQueue`, while agent replies
enter the conversation only through the temporary `say` CLI. Every lane has a
stable id, its own turn/observer state, lane-tagged host events, and an
append-only `oar-voyage/1` JSONL capture. The current renderer keeps the
single-lane launch flow for compatibility; fleet callers can use the explicit
IPC/API operations described below.

The complete OAR event stream remains visible in the Activity panel: Friendly
is the default semantic timeline and folds each tool call into one running/done
row; Raw keeps every OAR event one-for-one with its complete JSON available on
expansion.

Where the cockpit is headed — the ambitious dogfooding-and-verification
feature set and its build order — lives in [`ROADMAP.md`](ROADMAP.md).

## Relationship to oar

This repository consumes the published `@botiverse/oar` package from npm; it is
not part of the oar workspace. oar is a devDependency on purpose: it ships
ESM-only, and the main/preload bundles are CommonJS, so electron-vite must
bundle it into `out/` rather than externalize it as a runtime `require`. When the cockpit exposes a gap in oar, fix and
release oar first, then bump the dependency here. For local iteration against
an unreleased oar checkout, `pnpm link ../oar/packages/oar` (after building it)
and unlink before committing; CI only knows the published version.

## Run

```sh
pnpm install
pnpm dev
```

Choose an available runtime, optionally provide its native model identifier,
and choose an existing working directory. Closing the window disposes the
all lanes and removes their temporary `say` bridges. Voyage files default to
`.coxswain/voyages` in the working directory; set `COXSWAIN_VOYAGE_DIR` to
choose another directory.

## Fleet and voyage foundations

The main-process host exposes these lane-scoped operations:

- `launch({ runtimeId, cwd, laneId? })` starts one lane and allocates a
  `lane-N` id when `laneId` is omitted.
- `launchFleet({ lanes })` reserves all ids and starts lanes concurrently; a
  failed launch rolls back the whole fleet.
- `fleet()` returns stable lane identities, current observer views, and active
  turn ids. `submit`, `abort`, and `closeLane` accept a lane id (the omitted
  lane on `submit`/`abort` targets the most recently launched lane for legacy
  callers).
- `subscribeLane(id, listener)` receives only that lane's events; the normal
  host subscription receives every event with its `laneId` attached.

Every launched lane gets one voyage file. Its four record kinds are owned by
`@botiverse/oar`: a header, each human submission, each untouched public
`SessionEvent`, and a final end marker. The recorder buffers events around an
asynchronous steer/queue decision so a submission is always written first;
vendor-native raw output and debug logs remain separate concerns.

## Checks

```sh
pnpm check
pnpm test
pnpm build
```

`pnpm check` (typecheck + oxlint) must be green before every commit.

The deterministic screenshot smoke needs an X display on Linux and uses the
CommonJS launcher documented in `design/README.md`:

```sh
xvfb-run -a pnpm smoke
```

The screenshot is written to `artifacts/coxswain-smoke.png`. CI runs the same
sequence on Linux and uploads the screenshot as an artifact.

`experiments/say-bridge.ts` is a manual, token-burning check that the `say`
bridge delivers through the `OAR_SAY` indirection against a real runtime; run
it with `pnpm tsx experiments/say-bridge.ts` and read its `OBSERVED` header.

## Manual dogfood path

1. Launch a locally logged-in Claude installation.
2. Send a prompt and verify that Friendly Activity uses semantic action labels
   while Raw shows the complete event stream one-for-one.
3. Verify that messages emitted with `say` appear in the conversation.
4. Send another input while the turn runs and inspect its steered/queued marker.
5. Watch the status lamp follow the OAR observer, then abort the turn.
