# Atelier

Atelier is the estate's media MCP: a standalone Rust MCP server that
generates image, video, and music. It drives ComfyUI through a backend
trait, so the engine stays swappable and no tool surface changes when
it is.

Three lanes, one per model in the estate roster:

- **Iris** — image (Illustrious XL; txt2img and img2img)
- **Vera** — video (LTX-Video)
- **Aria** — music (ACE-Step)

## Lane profiles

One server, two profiles, selected by the running instance's
configuration. A caller cannot switch profiles per request.

- `qompass` — SFW work; Qompass output roots and logs; the
  fleet-facing profile.
- `ntranch` — adult work for Strobe; NTRanch output roots and logs.

The profiles' input, output, and log destinations are disjoint and
non-nested; configuration canonicalizes paths and fails closed on
overlap. Every result, provenance manifest, and log line carries its
profile stamp. Split servers were considered and set aside: a content
rating is a property of the request, not the engine.

## Lifecycle and GPU policy

Atelier owns the engine lifecycle. Jobs are serialized; ComfyUI
starts on demand, pinned to the RTX A6000, loopback only; a free-VRAM
admission check runs before each job; atelier stops only the engine it
started, after the job. It never stops, restarts, or reconfigures
rose, which shares the GPU.

## Stage state

- **S1 (image lane)** — brief written; the build waits on the fathom
  Ghidra G1 job releasing the primo Claude focus. S1 ships four tools
  (`atelier_status`, `atelier_models`, `atelier_txt2img`,
  `atelier_img2img`) and admits Iris only.
- **S2 (video)** and **S3 (music)** — planned; Vera and Aria are not
  reported as available until their stages land.

Acceptance for S1 is operational: the three Strobe gen-test scenes
regenerated through the public `atelier_img2img` tool under the
ntranch profile, with provenance, unchanged source hashes, and proof
that no output entered Qompass roots. Aesthetic acceptance stays with
the reviewer, not the worker.

## Documents

- [STUDY.md](STUDY.md) — the evidence: ComfyUI API facts, the model
  census, and the license filter that chose Iris.
- [PROGRAM.md](PROGRAM.md) — the design: profiles, GPU policy, stage
  plan, decision log, and the boundary with lumen (atelier generates
  flat media; lumen transforms sprites to 3D; the fold of atelier
  into lumen is a planned, trigger-gated convergence).
- [BRIEF-S1.md](BRIEF-S1.md) — the dispatch-ready S1 build brief.

## License

Apache-2.0. See [LICENSE](LICENSE).
