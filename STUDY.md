# Atelier — STUDY (2026-10-20)

## Commission and fork

Matt commissioned a local media-generation capability for Strobe and
then ruled the architectural fork on 2026-10-20:

- **Standalone media MCP — ADOPTED.** The estate gets its own small
  Rust MCP server, working name **atelier** (confirmed by Matt's
  pick), rather than treating ComfyUI as the estate-facing solution.
- **ComfyUI is an engine, not the product surface.** Atelier owns the
  MCP contract, model roster, workflow templates, lifecycle, output
  provenance, and GPU policy. ComfyUI sits behind a backend trait so
  a later native or alternative engine does not change atelier's
  tools.
- **Twilight is a consumer, not the host.** Matt also ruled that
  twilight must be able to do image generation. Twilight's Slice B
  Create lanes will consume atelier as an MCP client; atelier does
  not absorb twilight's studio, gallery, or client wiring.
- **One server, two lane profiles.** A later ruling the same day
  (recorded in PROGRAM.md's decision log) gives atelier two
  config-selected profiles, `qompass` and `ntranch`, with separate
  content classes, output roots, and logs. The profile is an
  instance configuration, not a per-request switch and not a second
  server.

This is the estate separation already used elsewhere: lookers and
actors stay separate (fathom/slipway, lumen/tackle, vongola/press).
Atelier is the media actor; consumers decide what to ask it to make.

## The media roster (ratified names)

Atelier addresses models by the ratified letter-keyed roster, not by
checkpoint filenames:

| Key | Capability | Model / local state |
| --- | --- | --- |
| `iris` | Image | Illustrious XL v1.1, an SDXL checkpoint, at primo `~/models/strobe-gen/`; verified SHA-256 begins `536863e9` and ends `8156c`. |
| `vera` | Video | LTX-Video, present in primo's Hugging Face cache as `models--Lightricks--LTX-Video`; the exact cached revision and ComfyUI-consumable file layout still need an S2 audit. |
| `aria` | Music | ACE-Step, present in primo's Hugging Face cache as `models--ACE-Step--Ace-Step1.5`; the exact cached component set and runtime route still need an S3 audit. |

For S1, only `iris` is admitted. Its full expected hash, canonical
filename, and optional display symlink belong in atelier's checked
configuration; a hash mismatch is a fail-closed condition, not a
warning.

Vera and Aria are named in the program now so the backend and job
model do not become image-only by accident. Naming them does not
claim either lane is implemented.

## Engine reality on primo

- Matt installed ComfyUI himself from the AUR: package version
  **0.39.0**.
- A separate engine bring-up is pinning the AUR layout, model path
  wiring, launch command, port, GPU selection, and a tame smoke in
  primo `~/ComfyUI-LAUNCH.md`.
- At the time this study was written, that launch document had not
  yet landed. Therefore this study does **not** assert the AUR
  executable name, install root, extra-model-path mechanism, port,
  startup time, or smoke result. BRIEF-S1 makes the completed,
  Pax-reviewed launch document a dispatch precondition.
- The operating facts atelier will require the bring-up to pin are:
  loopback-only bind; A6000 selection with
  `CUDA_DEVICE_ORDER=PCI_BUS_ID` and `CUDA_VISIBLE_DEVICES=1`;
  Iris visible to ComfyUI without copying the 6.9 GB checkpoint;
  clean shutdown that releases VRAM; and no alteration of rose.

## ComfyUI HTTP API — primary-source facts

Primary sources consulted 2026-10-20:

- ComfyUI official documentation, “ComfyUI Server Routes (HTTP and
  WebSocket API)”: https://docs.comfy.org/development/comfyui-server/comms_routes
- ComfyUI official documentation, “API Examples”:
  https://docs.comfy.org/development/comfyui-server/api-examples

The routes atelier's v1 backend needs are defined by ComfyUI's
`server.py` route table:

| Route | Method | Established behavior |
| --- | --- | --- |
| `/prompt` | POST | Validates and queues an API-format workflow; success returns a `prompt_id` and queue `number`; validation failure returns `error` and `node_errors`. |
| `/prompt` | GET | Returns current queue status and execution information. |
| `/queue` | GET / POST | Reads queue state; POST manages queue operations such as clearing pending/running work. |
| `/interrupt` | POST | Stops the currently executing workflow. |
| `/history` | GET | Returns execution history. |
| `/history/{prompt_id}` | GET | Returns history and outputs for one prompt. |
| `/upload/image` | POST | Uploads an image into ComfyUI's input side for workflows such as img2img. |
| `/upload/mask` | POST | Uploads a mask. Not needed by S1, but part of the same input family. |
| `/view` | GET | Retrieves a file using the filename, subfolder, and type recorded in history output. |
| `/system_stats` | GET | Reports system information including devices and VRAM information. |
| `/object_info` | GET | Returns definitions for all node types. |
| `/object_info/{node_class}` | GET | Returns definitions for one node type; useful for preflight checks. |
| `/models/{folder}` | GET | Lists models in a specific ComfyUI model folder. |
| `/free` | POST | Frees memory by unloading specified models. |
| `/ws` | WebSocket | Streams status, execution, progress, executed, and error messages. |

### Workflow JSON shape

`/prompt` takes a JSON object whose `prompt` member is an
**API-format workflow graph**, optionally with a `client_id` and a
client-supplied `prompt_id`. The graph is a map from node ID to an
object with:

- `class_type`: the ComfyUI node class;
- `inputs`: literal input values and graph links.

A graph link is a two-element array: `[source_node_id, output_index]`.
The official example's SDXL/SD graph shape is the familiar chain:
checkpoint loader → CLIP text encoders and latent image → sampler →
VAE decode → save image. Atelier will keep two versioned templates
for S1 — txt2img and img2img — and patch only named inputs (model,
prompt text, dimensions, seed, sampler controls, input image, and
output prefix).

The ordinary ComfyUI **UI workflow JSON is a different shape** (it
carries canvas/layout state) and is not interchangeable with the API
format. Atelier must never accept a UI-format export and guess at a
conversion.

### Completion and output retrieval

The official recommended pattern is:

1. Open `/ws?clientId=...` if live execution messages are wanted.
2. POST the API graph to `/prompt`.
3. Wait until the matching `executing` message reports a null node,
   or poll `/history/{prompt_id}`.
4. Read output descriptors (`filename`, `subfolder`, `type`) from the
   history record.
5. GET each output through `/view`.

Atelier's S1 client may poll history instead of holding a WebSocket;
that is an implementation choice, not a different contract. It must
have a bounded timeout and classify ComfyUI validation errors,
execution errors, missing history, missing output files, and engine
disappearance separately.

### Upload boundary

The official route table establishes `POST /upload/image`, but the
route summary does not spell out every multipart field and overwrite
rule. Atelier's implementation must verify the exact multipart
contract against the installed 0.39.0 server during its first
fixture test, using a harmless test image, before the acceptance
run. It must not treat a third-party client's field choices as
proof. The workflow's Load Image input must reference the name the
server actually returns or records for the upload.

## Video lane evidence (S2, not S1)

Lightricks' official LTX-Video README says, in substance, that the
project recommends its ComfyUI workflow for best results while also
shipping an `inference.py` local route for text/image-conditioned
video generation. The official README is at:
https://raw.githubusercontent.com/Lightricks/LTX-Video/main/README.md

That establishes two real candidate routes:

1. **ComfyUI LTX workflow** — preferred starting point because it
   reuses atelier's lifecycle, queue, output, and provenance model.
2. **Local LTX inference runner** — a separate backend adapter if the
   cached weights, node availability, version, or output quality make
   the ComfyUI route the wrong fit.

What is **not** established yet:

- which LTX revision and files are actually in primo's HF cache;
- whether those files are directly consumable by the AUR ComfyUI
  installation or require the official ComfyUI-LTX integration and a
  different layout;
- which workflow version will be pinned;
- the A6000 VRAM/time envelope for the frame counts Strobe or
  twilight will request.

S2 therefore begins with a cache and workflow audit, not with an
assumption that “the model is in the cache” means “ComfyUI can load
it as-is.”

## Music lane evidence (S3, not S1)

ACE-Step 1.5 is present as a local HF cache entry, and the upstream
project family provides non-ComfyUI runtime surfaces as well as
ComfyUI-oriented routes through node packages. This study does not
promote a route from secondary summaries.

S3 must first establish, from the installed cache and the upstream
project's own documentation/source:

- exact model components present (base model and any language-model
  companion), revision, disk layout, and license record;
- whether the estate's AUR ComfyUI can consume Aria through an
  audited node package without making atelier depend on an
  unreviewed custom-node supply chain;
- whether ACE-Step's own local API/CLI is the cleaner second backend;
- expected VRAM and generation-time bounds on the A6000;
- output format, metadata, and provenance fields.

Only then does S3 choose **ComfyUI backend** versus **native ACE
backend** behind atelier's same backend trait.

## Design consequences

1. **Templates, not arbitrary graphs, in S1.** Atelier does not expose
   a “run any ComfyUI workflow JSON” tool. Versioned SDXL templates
   make validation, tests, reproducibility, and provenance possible.
2. **The backend owns transport; the server owns policy.** The
   ComfyUI adapter knows `/prompt`, `/history`, `/upload/image`, and
   `/view`. Atelier owns model admission, parameter bounds, local
   path rules, serialization, lifecycle, output manifests, and
   errors.
3. **Lifecycle is part of correctness.** A generation that leaves a
   stray engine holding A6000 VRAM has not completed cleanly, even if
   the image is good.
4. **Profile separation is architectural.** Atelier runs under one
   config-selected lane profile: `qompass` for SFW work or `ntranch`
   for adult Strobe work. The profile fixes the content class,
   input/output roots, and log destination; a job cannot cross into
   the other profile's roots. Project-specific content decisions
   still belong to the consumer, but lane separation is enforced by
   atelier's configuration and path machinery rather than by caller
   discipline.
5. **Unknowns stay visible.** AUR layout, upload multipart details,
   cached Vera/Aria layouts, and later-lane workflow versions are
   pin-down tasks with owners and stages — not facts to be smoothed
   over in this study.
