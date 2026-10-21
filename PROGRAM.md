# Atelier — PROGRAM (2026-10-20)

## Identity

**atelier** — the estate's standalone media-generation MCP server,
written in Rust. It gives estate consumers one governed surface for
image, video, and music generation on primo, while keeping the
underlying engines replaceable.

Matt ruled the placement on 2026-10-20: a standalone media MCP, not
a ComfyUI skin and not a feature folded into twilight or lumen.
Atelier is infrastructure. It does not become a studio UI, a game
asset manager, or the authority for a project's specific content
decisions.

Atelier ships as **one server with two lane profiles**, selected in
configuration:

- `qompass` — SFW work; Qompass input/output roots and logs; the
  profile used by the fleet and registry-facing consumers.
- `ntranch` — adult work for Strobe; NTRanch input/output roots and
  logs, never mixed with Qompass outputs.

A profile is configuration for a running atelier instance, not a
second server and not a field a tool caller can switch per request.
It fixes the content class, roots, and log destination. One binary
and one host-wide engine-lifecycle authority serve either profile,
so two profiles never become two engines contending for the A6000.

- Repo: primo-local `~/atelier` until Matt gives the GitHub word.
- Registry: `ai.qompass/atelier` record prepared and **held**;
  publication waits for Matt's word after S1 gates.
- Packaging: Nix flake (estate container rule: no Docker/Podman).
- Protocol: estate MCP conventions — stdio, legacy `initialize`
  era like fathom/phlow/twilight, correct `serverInfo` from the
  first commit (name `atelier`, title `Atelier`, version from
  Cargo).
- Style: Tiger Style Rust; fail-closed TOML configuration; tests
  roughly half validation / half adversarial; every stage gated by
  Pax before the next begins.

## Consumers and dependency direction

| Consumer | Relationship |
| --- | --- |
| **Strobe** | First consumer, through the `ntranch` profile. Uses the image lane for game art and level art. Its first acceptance workload is the three-scene regeneration test defined in BRIEF-S1. |
| **Twilight** | Matt explicitly wants twilight to be able to generate images. Twilight's **Slice B Create** lanes will be a thin MCP client over atelier's image/video/music tools, consuming **both** profiles: `qompass` for SFW studio work and `ntranch` when it is acting for Strobe's adult generation. Twilight keeps its conversations, Create surface, and gallery; gallery items are backed by files and manifests produced by atelier under the corresponding profile. Twilight does not own the engine lifecycle or GPU policy. |
| **Lumen-adjacent media work** | May call atelier when a workflow needs generated media. Lumen does not host generation and does not absorb atelier. |
| **Phlow workflows** | May orchestrate atelier tools like any other estate MCP tools, subject to the consumer project's own approvals and policy. |

Dependency rule, mirroring “lumen consumes, never hosts”:

```text
Strobe ───────┐
twilight ─────┼──> atelier ──> media backend ──> local engine
lumen work ───┘
```

The arrow never reverses. Atelier does not call twilight, Strobe, or
lumen; it does not import their state; and a change inside atelier's
engine layer must not require a change to a consumer's gallery or
project model.

### Twilight Slice B note

When twilight reaches Slice B, its Create work splits cleanly:

- **Twilight B owns:** the Create UI/tool presentation, thin MCP
  client calls to the atelier instance configured for the required
  profile, conversation/project context, gallery records and
  display, and any user-facing queue presentation.
- **Atelier owns:** the active lane profile, model admission,
  workflow templates, engine start/stop, GPU contention, generation
  execution, output files, hashes, and provenance manifests.
- **Not in atelier's S1:** twilight's client wiring or gallery. That
  is twilight B work, separately briefed after atelier S1 gates.

## Design principles

1. **One server, two profile-enforced lanes.** The active profile
   is chosen by configuration and fixes the content class
   (`qompass` = SFW, `ntranch` = adult), the input/output roots, and
   the log destination. Tool calls cannot override it. Atelier still
   does not decide a consuming project's specific content rules;
   it enforces the lane boundary mechanically so outputs and logs
   cannot mix by caller error.
2. **Roster names are the public model identity.** Tools accept
   `iris`, `vera`, and `aria` as those lanes land. Checkpoint
   filenames, cache paths, and engine-specific model identifiers are
   configuration details.
3. **Templates before arbitrary power.** S1 exposes fixed, versioned
   txt2img and img2img templates. A general “submit any workflow”
   tool is a non-goal until a later stage proves it can be admitted
   without losing validation and reproducibility.
4. **One GPU, explicit contention.** Atelier serializes its own jobs
   and treats rose as a neighbor with priority of tenure: atelier
   never stops, restarts, or reconfigures rose, never uses the other
   GPU as a silent fallback, and stops its own engine after a job.
5. **Outputs are evidence.** Every successful job returns local file
   paths plus a provenance manifest: active profile, roster key,
   checkpoint identity and hash, workflow template version and hash,
   full bounded parameter set, seed, engine and version, source-image
   hash for img2img, output hashes, dimensions/format, and elapsed
   time. The profile name is stamped on every output manifest and
   every atelier log line.
6. **Fail closed.** Missing config, an unlisted model, a hash
   mismatch, a non-loopback engine bind, an unexpected GPU, an
   already-running engine atelier cannot account for, or an output
   path escaping its root all refuse the operation with a named
   error.
7. **No telemetry, no cloud fallback.** If the local engine cannot
   do the job, atelier reports that. It does not send prompts,
   source images, or outputs to an external service.

## Architecture

```text
MCP tools (Rust)
  │
  ├─ config + model admission
  ├─ job serializer / lifecycle lease
  ├─ provenance + output store
  └─ MediaBackend trait
       ├─ S1: ComfyUI backend
       ├─ S2: ComfyUI LTX workflow or local LTX adapter
       └─ S3: ComfyUI ACE adapter or native ACE adapter
```

The backend trait is deliberately narrow: health, model inventory,
submit a prepared job, await/collect outputs, interrupt the current
job, and release engine resources. Workflow construction stays in
lane modules above the trait so a future backend is not forced to
pretend it is ComfyUI.

### Configuration

Configuration is TOML, loaded from `$ATELIER_CONFIG` or
`~/.config/atelier/atelier.toml`. The active profile is a required
top-level value with no default:

```toml
profile = "qompass" # or "ntranch"
```

Each profile has its own configuration block naming its staged-input
root, output root, and log file/destination. The two profiles' roots
must be disjoint and neither may be nested inside the other;
configuration validation canonicalizes the paths and refuses overlap,
a missing profile block, an unknown profile, or any attempt to fall
back to shared output roots. A tool call receives the active profile
from the server instance and cannot select or alter it.

S1 configuration also names at least:

- atelier state and workflow-template roots (state may be shared for
  the host-wide lifecycle lock; generated outputs and logs may not be
  shared across profiles);
- the ComfyUI launch source pinned by `~/ComfyUI-LAUNCH.md`
  (executable/entry point and working directory, without a shell);
- loopback host and port (non-loopback values are refused);
- the fixed GPU selection environment and expected A6000 identity;
- startup timeout, job timeout, and free-VRAM admission threshold
  measured during engine bring-up;
- Iris's canonical checkpoint path and full SHA-256;
- parameter ceilings for dimensions, steps, batch, prompt length,
  and source-image bytes.

Secrets are not part of atelier's v1 shape: the engine is local and
loopback-only, and no credential belongs in a workflow or manifest.

## Engine lifecycle and GPU contention policy

This policy is normative for every lane.

1. **On-demand start.** Atelier starts ComfyUI only when a generation
   tool needs it and no accountable engine session is already owned
   by atelier.
2. **Pinned device.** The launch environment sets
   `CUDA_DEVICE_ORDER=PCI_BUS_ID` and `CUDA_VISIBLE_DEVICES=1`, the
   estate's A6000 selection on primo. Before submitting work,
   atelier verifies through ComfyUI system information that the
   visible device is the expected A6000 class device. A mismatch
   aborts the job.
3. **Loopback only.** ComfyUI is launched for `127.0.0.1` only.
   Atelier never exposes the engine on a LAN or tailnet address.
4. **Serialized jobs.** One atelier generation runs at a time. Other
   atelier calls wait behind a bounded queue or receive a named
   busy/queue-limit error; they never launch a second engine.
5. **Rose is never managed.** Atelier does not start, stop, restart,
   signal, or reconfigure rose. It does not call rose to ask
   permission; its protection is conservative admission and prompt
   release of its own engine.
6. **Admission check.** Before launch, atelier checks free VRAM on
   the selected GPU against the configured Iris threshold. If there
   is not enough room, the job is refused or queued without launching
   ComfyUI. Atelier does not evict another process's model to make
   room.
7. **No silent retry on OOM.** A CUDA out-of-memory or engine crash
   ends the job, triggers cleanup, and returns the evidence. Atelier
   does not loop retries against a busy GPU.
8. **Stop after the job.** After outputs and history have been
   collected, atelier stops the ComfyUI child it started and waits
   for exit, so VRAM returns to the host pool for rose and other
   work. If it attached to an engine it did not start, it must not
   kill that engine; S1's default is to refuse to attach to an
   unowned engine unless Pax changes that rule at the gate.
9. **Interruption cleans up.** On job timeout or cancellation,
   atelier calls ComfyUI `/interrupt`, collects any partial-output
   state honestly, then stops its owned engine.
10. **Contention is reported.** Status exposes the active profile,
    engine ownership, current job, queue depth, selected device, and
    the last cleanup result. Every status event and job log line
    carries the profile. A failed cleanup is a visible degraded
    state, not a log line that can be missed.

This policy favors predictable coexistence over throughput. Batch
efficiency work may revisit stop-after-job only with measurements
and Pax approval; it is not an S1 optimization to sneak in.

## Surface by stage

### S1 — image lane (Iris)

Public tools:

- `atelier_status` — active profile, server, config, backend,
  lifecycle ownership, selected GPU, queue/current job, model
  admission result, and last job/cleanup summary.
- `atelier_models` — admitted roster entries for implemented lanes,
  with capability, backend readiness, checkpoint identity, and
  pinned hash status. In S1 this is `iris` only; unimplemented
  roster names are not dressed up as available.
- `atelier_txt2img` — bounded SDXL text-to-image through the pinned
  txt2img template.
- `atelier_img2img` — bounded SDXL image-to-image from a source image
  under a configured input root, uploaded through ComfyUI and run
  through the pinned img2img template.

Generation calls are synchronous from the MCP caller's perspective,
with bounded timeouts. They return the atelier job ID, ComfyUI
prompt ID, output paths, hashes, dimensions, parameters actually
used, and provenance manifest path.

**S1 acceptance:** after unit/integration gates and a tame smoke,
atelier is configured for the **`ntranch` profile** and regenerates
the three ModificationApp test scenes named by primo
`~/strobe-gen-test/REGEN-BRIEFS.md` through `atelier_img2img`,
producing a source-vs-regen set with a manifest for Pax/Matt review.
All acceptance outputs and logs land under the NTRanch profile roots.
Details and pass conditions are in BRIEF-S1.

### S2 — video lane (Vera)

Planned surface, not S1 scope:

- `atelier_txt2video`
- `atelier_img2video`
- asynchronous job status/cancel tools if measured runtimes make
  S1's synchronous shape untenable.

S2 begins with the STUDY's audit: cached LTX revision/files,
official workflow or local runner viability on the installed AUR
ComfyUI, A6000 time/VRAM envelope, and output/provenance format. The
preferred first route is the official ComfyUI workflow; the local
LTX inference runner remains the evidence-based fallback behind the
same backend boundary.

### S3 — music lane (Aria)

Planned surface, not S1 scope:

- `atelier_txt2music` (final name settled in the S3 brief)
- duration, lyrics/tags, seed, and output controls only after the
  upstream route's real parameter contract is established.

S3 begins with the STUDY's route decision: audited ComfyUI node
backend versus ACE-Step's own local backend. It does not install a
custom-node package merely because one exists; supply-chain and
version evidence are part of the stage gate.

### Later, only on evidence and Matt's word

- masks/inpainting and ControlNet-style conditioning;
- LoRA admission and character-consistency workflows for Strobe;
- batch generation;
- additional checkpoints under the same hash-admission rule;
- a general workflow-submission tool;
- twilight's own Slice B client/gallery work (twilight's program,
  consuming this server).

## Stage gates

Every stage follows the same pattern:

1. Written brief with explicit non-goals and acceptance evidence.
2. One primo worker build at a time, under a per-job profile.
3. `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`,
   and `cargo test` with exact counts, roughly half validation /
   half adversarial.
4. Stdio smoke against the real server binary.
5. Real-engine smoke on primo with the lifecycle policy observed:
   A6000 selected, rose untouched, owned engine stopped, VRAM
   released.
6. Pax gate. Only then does the conformance entry, SERVERS.md row,
   progress-board movement, or next stage follow.

Registry preparation may happen while S1 is in review, but the
`ai.qompass/atelier` record is **held** until Matt authorizes
publication after S1 gates.

## Decision log

- **2026-10-20 — Standalone server.** Matt ruled for a standalone
  media MCP (atelier) rather than making ComfyUI the estate surface
  or folding generation into twilight. ComfyUI is the v1 engine
  behind a backend trait.
- **2026-10-20 — Twilight consumes atelier.** Twilight's Slice B
  Create lanes will call atelier as a client for image generation
  (and later video/music). Dependency runs twilight → atelier,
  never the reverse.
- **2026-10-20 — One server, two lane profiles.** Atelier ships as
  one server with config-selected profiles: `qompass` for SFW work
  and `ntranch` for adult Strobe work. The profile fixes content
  class, output roots, and log destination, and its name is stamped
  on every output and log line. The split-servers alternative was
  considered and set aside on Pax's recommendation: rating is a
  property of the request, not the engine, and one lifecycle avoids
  GPU contention between two servers.

## Open questions for Matt

- **Q-ATELIER-1 — GitHub home.** Create `qompassai/atelier` now, or
  keep the repo primo-local until S1 gates? Program default follows
  twilight/tackle: local-only until Matt's word. Does not block S1.
- **Q-ATELIER-2 — Registry timing.** Pax recommendation: prepare the
  `ai.qompass/atelier` record during S1 review and publish only on
  Matt's explicit word after the S1 gate.
- **Q-ATELIER-3 — Attached-engine rule.** S1 refuses to use a
  ComfyUI instance atelier did not start, because it cannot safely
  own cleanup. If Matt wants atelier to reuse a ComfyUI he already
  has open, that needs a separate rule for “attach but never stop”
  and a decision about whether `/free` may be called. Not S1's
  default.
- **Q-ATELIER-4 — Output access.** S1 returns primo-local paths and
  hashes, which fit Strobe and twilight on the same host. Does any
  near-term consumer need bytes or MCP resources returned directly?
  If so, that is a later surface decision, not an S1 assumption.
- **Q-ATELIER-5 — Model admission.** S1 admits Iris only, pinned by
  full SHA-256. Future checkpoints and LoRAs need a named admission
  rule: who approves a model, and is a license record required in
  config before first use? Pax recommendation: yes — name, source,
  license posture, and hash are recorded before admission.
- **Q-ATELIER-6 — Commercial-use legal read.** The model hunt chose
  Iris on its card's SDXL/CreativeML Open RAIL++-M posture but
  flagged a dedicated license read as still owed before Strobe's
  commercial release. Atelier can gate and test locally; production
  use of outputs waits for that separate read.
- **Q-ATELIER-7 — S2/S3 route choices.** If the audits contradict
  the preferred routes (ComfyUI for Vera; unresolved for Aria),
  Pax will bring the evidence and the route choice back rather than
  silently adding a second runtime.

## Boundary with lumen (Pax, 2026-10-20, from Matt's overlap question)

Atelier generates flat media (image, video, audio) from prompts and
references. Lumen is the sprite-to-3D character factory: identity
lock, rigging, blendshape expressions, engine export; its renders are
verification outputs of that pipeline, not generation. The two meet
at exactly one handoff: lumen may ingest atelier-generated sprites
as sources. Atelier never rigs, models, or exports 3D. Lumen never
grows a diffusion or generation backend. Twilight's Create lanes sit
above both as the studio surface.

## Planned convergence with lumen (Matt's direction, 2026-10-20)

Matt's stated end-state: atelier may eventually fold INTO lumen, so
lumen offers flat generation (image, and later video/audio) alongside
its sprite-to-3D transformation work. This is a planned convergence,
not a current merge, and it is trigger-gated:

1. Atelier S1 gated and in real use, with lumen consuming atelier as
   a client for sprite sources (the handoff exercised in production).
2. Lumen Stage 4 remediation closed; lumen's own house in order.
3. A real consumer workflow (Strobe or twilight) routing through
   both in one pass.

Until the triggers are met: separate servers, one handoff, and no
design decisions that would make the fold a rewrite — the backend
trait, the config-selected lane profiles, and the tool shapes are
deliberately fold-compatible. On a fold, the profile system (qompass
/ ntranch roots, logs, stamps) survives intact inside lumen's
generation module; a fold that dissolves the lane wall is not the
planned fold.

## Rulings (Matt, 2026-10-20)

- Q-ATELIER-1: GitHub home NOW — create qompassai/atelier.
- Q-ATELIER-2: Yes — registry record prepared during S1 review,
  published only on Matt's word after the S1 gate.
- Q-ATELIER-3: Agreed with Pax — S1 uses only an engine atelier
  started; attach-mode is a later, separate rule.
- Q-ATELIER-4: Agreed with Pax — S1 returns primo-local paths and
  hashes; bytes/resources are a later surface decision.
- Q-ATELIER-5: Yes, as amended by Matt the same day — admission
  requires name, source, and hash recorded before first use. The
  LICENSE posture is alert-not-deny: if no license record exists,
  atelier admits the model but raises an alert (warn-stamped log
  line, a license_alert field on results, and license: UNRECORDED
  stamped on output provenance manifests). Outputs from an
  unrecorded-license model are automatically not cleared for
  production, which is how Q-ATELIER-6 stays enforceable without a
  gate.
- Q-ATELIER-6: Agreed — the dedicated commercial-use license read is
  owed before Strobe's production release; it does not block S1.
- Q-ATELIER-7: Understood — route contradictions come back with
  evidence; no silent second runtime.
