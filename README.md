# Ouroboros Metal
### Edge-Native GPU Transcoding & Media Intelligence Pipeline

A deterministic, event-driven video pipeline that runs the entire OTT workload
— ingest, GPU ABR transcoding, HLS packaging, and playback — on a single
consumer machine, with the cloud reduced to a thin metadata plane.

Built on one axiom: **frames never leave VRAM.** Decode, scale, and encode all
execute on the GPU die; the pipeline's job is to stay out of the way.

---

## Proven Performance (measured, not projected)

| Metric | Result |
|---|---|
| 596s of 1080p source → full 3-variant ABR ladder | **111.8s (5.3× realtime)** |
| Chunk retry count across all verification runs | **0** |
| Concurrent NVENC encode sessions | 6 (2 workers × 3 variants) |
| Full stack cold boot | **11.3s** |
| Hardware | Single RTX 4070 Ti, consumer NVMe, WSL2 |

Every number above comes from an end-to-end verification gate on real media,
recorded in the project's state ledger with evidence.

## Architecture

**Control plane** — FastAPI on the host: streaming ingest, in-RAM `ffprobe`
audit (2MB head-slice with MOOV fallback), JWT minting, static HLS origin,
status API. Cloud contact is a single metadata upsert per job against a
serverless Postgres — never during processing.

**Compute plane** — Dockerized Celery workers on Redis: a content-addressed
chunk ledger (SHA256 CIDs) splits the source into 4-second chunks, each
transcoded through a single-decode CUDA graph — NVDEC → `scale_cuda` ×3 →
`h264_nvenc` ×3 — producing 1080p/720p/360p simultaneously without the frames
ever touching system RAM. Differential ledger healing makes re-runs
incremental. Manifest assembly emits standard HLS (master + variants,
MPEG-TS segments).

**Failure discipline** — unsupported inputs fail fast at audit with a legible
reason, not a retry storm. Enhancement stages are isolation-wrapped: they can
fail without taking the proven transcode path down.

## Design Principles

- **VRAM invariant:** one decode per chunk; no `hwdownload` chains, ever.
- **Determinism:** every stage has verifiable output; state lives in an
  append-only supersession ledger; one commit per ratified state.
- **Anti-entropy:** no frameworks where a filter graph will do; C/C++-backed
  runtimes only (ONNX Runtime over PyTorch); dependency count is a metric.
- **Edge-first economics:** the cloud stores metadata; silicon you own does
  the work.

## Roadmap (specced, in flight)

- **Scene intelligence:** native FFmpeg hardware scene-boundary detection
  into the chunk ledger.
- **Semantic search:** SigLIP embeddings via ONNX Runtime (CUDA) into
  `sqlite-vec` on local NVMe — sub-millisecond RAG moment search with zero
  cloud round-trips.
- **Smart verticalization:** TensorRT object detection driving in-VRAM 9:16
  crops.
- **10-bit ingest:** HEVC ladder / bit-depth conversion paths (currently
  fail-fast rejected with a clear reason).

## Genesis: the Cloud-Native Predecessor

Ouroboros Metal is a ground-up rebuild of a fully working cloud pipeline —
Payload CMS → GCS → Eventarc/Cloud Run orchestration → AWS Elemental
MediaConvert → S3/CloudFront behind a sub-millisecond JWT edge function.
Building it end-to-end surfaced the thesis: for this workload class, managed
cloud transcoding trades determinism and cost for convenience you don't need.
The edge rebuild kept the good ideas (event-driven design, atomic locks,
zero-trust tokens, telemetry hooks) and moved the physics on-die.

The original architecture is preserved on the [Genesis page](./index.html)
as documented lineage.

---

*Open-source release: targeted mid-August 2026. The repository ships with its
full engineering ledger — every defect, ruling, and verification gate that
produced the numbers above.*
