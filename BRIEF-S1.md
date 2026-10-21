# BRIEF-S1 — atelier image lane: Iris through the ComfyUI backend

Governed by PROGRAM.md; evidence and API grounding in STUDY.md (both
in this package). Scope: build atelier's S1 image lane only — a Rust
MCP server, two config-selected lane profiles, one ComfyUI backend,
and the `iris` model — and prove it on the Strobe regeneration test.

**Dispatch status: NOT DISPATCHED by this package.** Pax dispatches
this brief only when every precondition below is true. In
particular, **dispatch waits until the fathom Ghidra G1 job has
released the primo Claude focus and Pax has gated or explicitly
cleared that job**. One live primo Claude worker at a time. A
dispatcher must never kill, preempt, or work around the G1 focus
holder to start atelier.

## Preconditions (all must be verified, none assumed)

1. **Focus free.** `pax-fathom-ghidra-g1` is complete or explicitly
   stood down, its report has been dealt with by Pax, and the
   pax-focus state no longer names a live holder.
2. **Engine bring-up landed.** Primo `~/ComfyUI-LAUNCH.md` exists,
   has been reviewed by Pax, and pins the installed AUR ComfyUI
   0.39.0 layout: executable/entry point, working directory,
   loopback address and port, A6000 environment, Iris model wiring,
   startup/shutdown procedure, and a successful tame smoke with its
   workflow artifact. If any of those facts is missing, stop; S1 is
   not the job that guesses the engine layout.
3. **Acceptance inputs landed.** Primo
   `~/strobe-gen-test/REGEN-BRIEFS.md` exists and names exactly
   three scenes, their source files, prompts, negative prompts, and
   target settings. Source files exist under
   `/home/phaedrus/strobe-gen-test/`; their hashes are recorded
   before the worker starts. If the brief file or a named source is
   absent, report blocked rather than selecting substitute scenes.
4. **Model identity known.** Iris's canonical checkpoint is present
   under `/home/phaedrus/models/strobe-gen/` and matches the full
   SHA-256 recorded in the dispatch packet / atelier config. The
   abbreviated fingerprint in STUDY.md is not sufficient for the
   worker's admission check.
5. **Per-job profile ready.** The dispatcher has created and audited
   the atelier S1 profile described at the end of this brief.

## Environment and repository

- Host: primo only.
- New repo: `/home/phaedrus/atelier` (local git, **no remote, no
  push** until Matt's GitHub word).
- Branch: `pax/atelier-s1`. Make the initial scaffold commit if the
  dispatcher has not already created the repo, then do all S1 work
  on this branch with local commits in logical slices.
- Stage/report root: `/home/phaedrus/atelier-s1/`; the completion
  report is `/home/phaedrus/atelier-s1/REPORT.md`.
- Engine: the AUR ComfyUI installation exactly as pinned by
  `/home/phaedrus/ComfyUI-LAUNCH.md`. Do not install, upgrade, or
  reconfigure ComfyUI; do not install custom nodes; do not use
  Docker/Podman; do not use sudo; do not change any service.
- GPU: A6000 only, with `CUDA_DEVICE_ORDER=PCI_BUS_ID` and
  `CUDA_VISIBLE_DEVICES=1`. Never fall back to the other GPU.
- Rose: read-only coexistence check only. Do not restart, signal,
  reconfigure, unload, or send generation traffic to rose.
- No external generation service and no network call except package
  tooling already required by the Rust/Nix build and loopback calls
  to ComfyUI.

## Deliverable 1 — Rust MCP server

Create crate/binary `atelier`, following twilight's estate pattern
for rmcp, server configuration, and stdio behavior:

- `serverInfo`: name `atelier`, title `Atelier`, version from Cargo,
  correct from the first commit.
- Legacy MCP `initialize` era, consistent with fathom/phlow/twilight.
- Fail-closed TOML config from `$ATELIER_CONFIG` or
  `~/.config/atelier/atelier.toml`. Missing, unreadable, or invalid
  configuration refuses startup with a named error; there is no
  permissive default config.
- Structured errors with stable machine-readable codes. Error text
  must distinguish configuration, profile, model admission, source
  path, engine lifecycle, workflow validation, execution, output,
  and cleanup failures.
- Nix flake with `packages.default` and `devShells.default`.
- README with build/run shape and a two-profile configuration
  example using placeholder roots. Apache-2.0 LICENSE copied from
  an estate sibling.
- No telemetry.

### Exact S1 tool list

The stdio smoke pins exactly these four tools:

1. **`atelier_status`** — no arguments. Returns active profile,
   server version, config status, backend kind, engine state
   (`stopped`, `owned_running`, `unowned_present`, `degraded`), the
   lifecycle owner, selected GPU identity/status, current job and
   queue depth, admitted models, and last job/cleanup result. This
   tool must not start the engine.
2. **`atelier_models`** — optional `lane` argument (`image`,
   `video`, or `music`); unknown values are refused. Returns only
   models whose lane is implemented and admitted. In S1 that is
   `iris` (image) with its backend readiness and pinned-hash result.
   It must not claim `vera` or `aria` is available. This tool must
   not start the engine.
3. **`atelier_txt2img`** — generates one image with `iris`.
4. **`atelier_img2img`** — generates one image with `iris` from one
   admitted source image.

No other public tools in S1.

### Generation arguments and bounds

For both generation tools:

- `model` is required and must be exactly `iris`.
- `prompt` is required; `negative_prompt` is optional.
- `width`, `height`, `seed`, `steps`, `cfg`, `sampler_name`, and
  `scheduler` are optional within the ceilings and allowed values in
  config / the pinned template. Defaults come from the versioned
  template and are echoed back as the parameters actually used.
- Batch size is fixed at 1 in S1. There is no caller-supplied output
  path, filename, workflow JSON, node graph, shell command, engine
  URL, GPU selector, or profile selector.
- Dimensions must be positive multiples of 8 and within the
  configured per-axis and total-pixel ceilings. Prompt and source
  sizes are byte-bounded by config.

`atelier_img2img` additionally takes:

- `source_path`: a local image path that must resolve under one of
  the active profile's configured allowed source roots;
- `denoise`: a finite number in `[0.0, 1.0]`, subject to the
  template's documented operating range.

Successful generation returns a structured object containing:
atelier job ID, active profile, ComfyUI prompt ID, model key,
parameters actually used, source SHA-256 (img2img), output path,
output SHA-256, output byte length, width/height/format, elapsed
milliseconds, and provenance manifest path.

## Deliverable 2 — two lane profiles (in scope, not a stub)

Atelier has one binary and one active profile per running instance.

### Profiles

- `qompass`: content class `sfw`; Qompass staged-input, output, and
  log destinations.
- `ntranch`: content class `adult`; NTRanch staged-input, output,
  and log destinations.

The profile comes only from the required top-level config value
`profile`. There is no default and no environment-variable or tool
argument override in S1. Changing profile means changing config
and starting a different atelier instance under the same host-wide
lifecycle rules.

### Separation rules

- Each profile block names its own allowed source roots, staged
  input root, output root, and log destination.
- Canonicalized Qompass and NTRanch output roots must be disjoint and
  non-nested. The same rule applies to staged-input roots and log
  destinations. Overlap is `profile_root_overlap` and refuses
  startup.
- Every path atelier writes is constructed beneath the active
  profile's roots using server-generated job IDs. Caller strings are
  never used as output path components.
- Source paths are canonicalized before use. A source outside every
  allowed root for the active profile, a symlink escape, a directory
  traversal, or a non-regular file is refused.
- Backend-returned output descriptors are untrusted. A filename,
  subfolder, or fetched result that would resolve outside the
  active profile output root is refused as
  `output_escape_refused`; atelier must not “helpfully” relocate it.
- The active profile string (`qompass` or `ntranch`) is stamped on:
  `atelier_status` and `atelier_models` results, every generation
  result, every provenance manifest, the acceptance manifest, and
  every atelier log line/event.
- ComfyUI's own working/output directories are engine internals.
  Atelier's contract outputs are the files it collects and writes
  under the active profile root, with hashes and manifest. The
  worker must account for any engine-side intermediate path in the
  report; an intermediate does not become a second deliverable root.

## Deliverable 3 — ComfyUI backend and lifecycle

Implement a narrow `MediaBackend` boundary and one S1
`ComfyBackend`. The MCP/tool layer must not contain raw ComfyUI URL
construction except through the backend.

### Backend operations

- health / system information;
- model-folder inventory for checkpoint admission checks;
- node-class preflight through `/object_info/{node_class}`;
- image upload through `/upload/image`;
- prompt submission through `/prompt`;
- completion collection through `/history/{prompt_id}` (history
  polling is the S1 baseline; WebSocket support is not required);
- output retrieval through `/view`;
- current-job interruption through `/interrupt`;
- queue inspection through `/queue` or GET `/prompt` for lifecycle
  and diagnostics.

Before the acceptance run, prove the installed server's exact
`/upload/image` multipart field and response contract with a small
harmless fixture and record it in the report. STUDY.md deliberately
does not invent that detail from the route name alone.

### Workflow templates

Keep two versioned API-format templates in the repo (exact file
names may follow the crate's asset layout, but the report must name
them):

- Iris txt2img: checkpoint loader → positive/negative CLIP encoding
  → empty latent → sampler → VAE decode → save image.
- Iris img2img: checkpoint loader → positive/negative CLIP encoding
  → uploaded-image load/encode → sampler at the requested denoise →
  VAE decode → save image.

Templates are JSON data with stable node IDs and named patch points.
Atelier validates that the JSON is API-format (a map of node objects
with `class_type` and `inputs`), patches only the allowed inputs,
and records template version + SHA-256 in every manifest. UI-format
workflow JSON is refused, not converted heuristically.

### Lifecycle

- Start ComfyUI on demand for a generation job using the launch
  facts pinned in `~/ComfyUI-LAUNCH.md`, without invoking a shell
  string from a tool call.
- Bind/use loopback only. Refuse a configured non-loopback engine
  address.
- Serialize generation with an atelier job lock. The host-wide
  lifecycle lease prevents two atelier instances under different
  profiles from owning engines at the same time.
- Before submission, verify the engine's selected device is the
  A6000 and the Iris checkpoint is visible/admitted.
- Record GPU memory before launch, after engine readiness, after
  collection, and after shutdown in the real-engine gate evidence.
- On success, failure, timeout, or interruption: collect what can
  be collected honestly, interrupt an owned running prompt when
  needed, stop only the ComfyUI process atelier started, wait for
  it to exit, and verify the process is gone. Never use a broad
  process-name kill.
- If an engine atelier did not start is already present, S1 does
  not attach to it and does not stop it. Generation returns a named
  unowned-engine error and status reports the condition. Reusing a
  user-opened ComfyUI is a later design decision (PROGRAM
  Q-ATELIER-3), not worker improvisation.
- If cleanup fails, status becomes degraded and the report says so
  plainly. A good image does not erase a failed cleanup gate.

## Deliverable 4 — provenance and logs

For each generation, write one JSON manifest beside the output (or
in a per-job directory under the active profile output root) with:

- atelier job ID and active profile;
- model key, checkpoint canonical identity, and full SHA-256;
- backend kind, ComfyUI version if the engine reports it, and
  ComfyUI prompt ID;
- workflow template name, version, and SHA-256;
- full parameters actually used, including seed and denoise;
- source path as an atelier-relative/root-labelled reference and
  source SHA-256 for img2img (do not copy credentials or unrelated
  absolute consumer paths into public summaries);
- output filename, byte length, SHA-256, dimensions, and format;
- start/end timestamps and elapsed time;
- lifecycle cleanup result.

Logs are append-only under the active profile's log destination.
Every log event includes timestamp, level, event name, and
`profile`. Job events also include job ID and prompt ID. Qompass and
NTRanch logs must never be the same file.

## Tests — roughly half validation, half adversarial

Report the actual counts by class; do not tune the count by
renaming tests after the fact.

### Validation coverage (expected shape)

- config loads for a valid `qompass` profile and a valid `ntranch`
  profile;
- model inventory admits the correctly hashed Iris fixture and
  reports video/music as unimplemented;
- txt2img template patches only named inputs and preserves graph
  links;
- img2img template patches source and denoise correctly;
- fake-Comfy HTTP fixture: successful submit → history → `/view`
  collection;
- upload fixture records the server-returned input name;
- history output descriptors become safe per-job output paths;
- status and models work while the engine is stopped and do not
  start it;
- manifests contain profile, model hash, template hash, parameters,
  source hash (img2img), and output hash;
- lifecycle fixture starts one fake engine session, serializes two
  jobs, and stops only the owned child.

### Adversarial coverage (expected shape)

- missing config, malformed TOML, unknown profile, missing profile;
- Qompass/NTRanch roots equal, nested, or made overlapping through
  `..` or a symlink;
- a tool call attempting to select the other profile (extra field /
  wrong schema) is refused;
- checkpoint missing, wrong hash, or a different admitted filename;
- UI-format workflow JSON; template with an unexpected node class,
  a missing patch point, or a graph link outside the node map;
- out-of-range dimensions, excessive total pixels, zero/negative
  values, non-finite or out-of-range denoise, oversized prompt,
  unsupported sampler/scheduler, and a model other than `iris`;
- source path traversal, absolute path outside allowed roots,
  symlink escape, directory as source, unsupported image format,
  and an oversized source file;
- Comfy validation failure (`error` + `node_errors`), execution
  failure, empty history, history for the wrong prompt, missing
  output descriptor, `/view` 404, and malformed JSON;
- fake backend returns `../`, an absolute path, or a symlink target
  outside the active profile output root — no bytes are written
  outside the root;
- engine already running but unowned; wrong GPU identity; low free
  VRAM; engine exits during startup or mid-job; job timeout;
- cleanup failure is surfaced as degraded status, not converted to
  success.

Use a local fake HTTP backend fixture for transport and error tests.
Do not make the unit suite depend on the real A6000 or on launching
the AUR engine.

## Real-engine gates (on primo, exact evidence in REPORT.md)

Run these after the automated suite is green:

1. **Stopped-engine read gate.** With no atelier-owned ComfyUI
   running, drive the built server over stdio: `initialize`,
   `tools/list` (exactly the four S1 tools), `atelier_status`, and
   `atelier_models`. Confirm no engine process was started.
2. **Qompass tame smoke.** Configure profile `qompass` with test
   roots under `/home/phaedrus/atelier-s1/smoke/qompass/`. Generate
   one non-adult 1024×1024 stage/background image through
   `atelier_txt2img`. Verify PNG decoding, dimensions, nonblank
   pixel variance, output hash, manifest profile = `qompass`, and
   clean engine shutdown / VRAM release.
3. **Profile separation smoke.** Configure profile `ntranch` with
   test roots under `/home/phaedrus/atelier-s1/smoke/ntranch/` and
   run the same tame image request through the tool layer or a
   test-only invocation of the same generation path. Verify its
   output and log are under NTRanch roots only and the Qompass test
   roots gained no new output or log lines. This is a separation
   proof, not the adult acceptance run.
4. **Img2img fixture smoke.** Use a harmless local test image under
   the active profile's allowed roots and run `atelier_img2img`
   end-to-end. This proves the multipart upload contract before any
   acceptance scene is touched.
5. **Coexistence evidence.** Record rose's loopback version/tags
   reachability before and after the S1 engine runs (metadata only;
   no rose generation), plus `nvidia-smi` memory snapshots showing
   the A6000 was selected and atelier's engine exited afterward.

## First acceptance — Strobe regeneration test (ntranch profile)

This acceptance is operational and evidentiary. It proves atelier can
perform the requested generations and preserve the set; Pax and
Matt judge visual quality separately.

### Inputs

- Governing scene list: `/home/phaedrus/strobe-gen-test/REGEN-BRIEFS.md`.
- Exactly the three scenes named there. No substitutions, no extra
  scenes, no prompt edits except mechanical placement into the tool
  arguments. If a brief omits a required prompt, source, or target
  setting, stop and report the missing item.
- Source assets remain read-only. Hash every source before and after;
  the hashes must match.

### Run configuration

Use a dedicated acceptance config with:

- `profile = "ntranch"`;
- allowed source roots containing the extracted test assets under
  `/home/phaedrus/strobe-gen-test/`;
- staged-input root, output root, and log destination under
  `/home/phaedrus/atelier-s1/acceptance/`, in separate NTRanch
  subdirectories named in the report;
- Iris pinned by the same full SHA-256 as the production sample.

### Procedure

1. Run all three scenes through the public MCP tool
   `atelier_img2img`, not by calling ComfyUI directly.
2. For each scene, preserve in its own directory:
   - the source image (copied without modification) or a clearly
     identified link-free duplicate whose hash equals the source;
   - the regenerated output exactly as collected by atelier;
   - the atelier provenance manifest;
   - a small `SCENE.md` naming the scene ID from REGEN-BRIEFS, the
     tool-call parameters actually used, source and output hashes,
     dimensions, elapsed time, and any deviation.
3. Create `/home/phaedrus/atelier-s1/acceptance/INDEX.md` listing
   all three pairs in REGEN-BRIEFS order, with relative links to
   source, regen, manifest, and scene note.
4. Create `/home/phaedrus/atelier-s1/acceptance/manifest.json`
   recording acceptance profile (`ntranch`), Iris hash, template
   hash, per-scene source/output hashes and paths, and the overall
   result. Every scene's individual manifest must also say
   `profile: ntranch`.
5. Validate every regen as a decodable image, check dimensions
   against its brief, and record a nonblank measure (for example
   pixel standard deviation). A black, empty, corrupt, or
   wrong-scene file is a failed scene, not a cosmetic issue.
6. Verify no acceptance output or log line was written under the
   Qompass smoke roots, and no Qompass output was written under the
   acceptance roots.

### Acceptance pass condition

S1's operational acceptance passes only if all three scenes complete
through `atelier_img2img`, the source-vs-regen set and manifests are
complete, sources are unchanged, profile separation holds, and the
owned ComfyUI process is stopped afterward. Visual quality is
recorded as **pending Pax/Matt review**, never as a worker verdict.

## Repository gates

All commands run in `/home/phaedrus/atelier` and are reported with
exact counts/output:

- `cargo fmt --check`
- `cargo clippy --all-targets -- -D warnings`
- `cargo test` (and `cargo test --all-targets` if the crate layout
  makes that a distinct run)
- `nix build`
- stdio smoke against the built binary (initialize + tools/list +
  status + models)
- the real-engine gates and acceptance procedure above

Before every commit, run `git diff --cached --stat` and confirm only
atelier S1 files are staged. Local commits only. **No push.**

## REPORT.md (required)

Write `/home/phaedrus/atelier-s1/REPORT.md` with:

- branch and commit hashes;
- files created/changed, by logical slice;
- exact fmt / clippy / test / nix results and the test count split
  (validation vs adversarial);
- stdio transcript summary and the exact four-tool list;
- engine bring-up facts consumed from `~/ComfyUI-LAUNCH.md` (AUR
  entry point, port, model wiring) and the proven `/upload/image`
  multipart contract;
- tame-smoke and profile-separation evidence, including output and
  manifest paths/hashes;
- GPU/rose coexistence snapshots and cleanup result;
- acceptance INDEX/manifest paths, per-scene source/output hashes,
  dimensions, elapsed times, and deviations;
- every failure encountered during the build and how it was resolved;
- open issues stated as issues, not buried in prose.

A task is not done until this report exists.

## Out of scope — do not build

- Vera/video tools, Aria/music tools, or their backend adapters.
- Twilight's MCP client wiring, Create surface, or gallery. That is
  twilight Slice B work after atelier S1 gates.
- Lumen changes or lumen-side media integration.
- Strobe game integration, Unity changes, or production asset work.
  The acceptance set is a test set; nothing from ModificationApp
  becomes a Strobe asset through this brief.
- Inpainting/masks, ControlNet, LoRAs, batch generation, additional
  checkpoints, model downloads, or a general arbitrary-workflow
  submission tool.
- Attaching to a user-opened ComfyUI, remote/LAN engine exposure, a
  second engine, or GPU fallback.
- Diver configuration, SERVERS.md, MCP conformance files, registry
  record preparation/publication, progress-board edits, or GitHub
  repo creation. Pax lands integration records after the gate.
- Any change to rose, the AUR ComfyUI installation, ComfyUI custom
  nodes, primo services, or the extracted source assets.

## Dispatcher notes — per-job permission profile

The dispatcher (Pax, at dispatch time) creates
`/home/phaedrus/.claude/atelier-s1-settings.json`, mode `600`, by
copying the current worker profile's allow list verbatim and keeping
every worker deny, including `Bash(git push:*)`. Add only the exact
shapes S1 needs, after auditing them against the installed profile
format. Expected additions are limited to:

- file tools within `/home/phaedrus/atelier` and
  `/home/phaedrus/atelier-s1`, plus read access to
  `/home/phaedrus/ComfyUI-LAUNCH.md`,
  `/home/phaedrus/strobe-gen-test/REGEN-BRIEFS.md`, the named source
  assets, and the Iris checkpoint metadata/hash;
- `cargo` build/test/fmt/clippy and `nix build` in the atelier repo;
- local git status/diff/log/add/commit/branch/switch in the atelier
  repo only;
- loopback `curl` to the ComfyUI address pinned by the launch doc
  and to rose's metadata endpoints for coexistence checks;
- the exact ComfyUI launch/stop commands from
  `/home/phaedrus/ComfyUI-LAUNCH.md`, `nvidia-smi`, `sha256sum`,
  `stat`, `file`, `jq`, `mkdir`, `cp`, and bounded `sleep`/`timeout`
  shapes needed for lifecycle waits.

No `paru`/`pacman`, no `pip`, no model download, no broad `python3`,
no `rm -rf`, no `git reset --hard`, no `git clean`, and no push
shape is added. If a needed command has no safe narrow shape, the
worker stops and reports the blocker; it does not widen its own
profile.

Register the dispatch in the worker watchdog as:

```text
atelier-s1|pax-atelier-s1|/home/phaedrus/atelier-s1/REPORT.md
```

Registration and direct liveness verification (session + process +
early output) are Pax/dispatcher duties at dispatch time. This brief
alone does not authorize bypassing the one-live-primo-worker rule.
