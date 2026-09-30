# SnapMind

**A local-first sensitive-text inspector and redactor designed for Snapdragon-powered HP PCs.**

SnapMind helps people review copied notes, support tickets, and document text before sharing it. It detects structured identifiers and context-dependent personal details, produces a redacted copy, and exposes the execution backend and measured local latency.

> Competition target: Snapdragon® AI Lab Build & Present Challenge – 2026. This repository began as an empty workspace; SnapMind is therefore a new project, not a modification of a pre-existing application.

## Problem statement

People routinely paste personal details into collaboration tools and AI services. Manual review is inconsistent, while cloud-only privacy scanners require sending the very text being inspected to another service. Existing regex-only tools also miss context-dependent details such as names and locations.

## Solution

SnapMind performs a hybrid local inspection:

- deterministic recognizers find emails, phone numbers, payment-card candidates, and IP addresses;
- a lightweight contextual token classifier identifies likely names and locations;
- a single `AIInferenceEngine` selects a verified QNN/NPU path only when a compatible model and provider load successfully;
- the UI reports `SNAPDRAGON NPU` or `CPU FALLBACK`, measured inference/load time, and whether inference needs a network.

The tool is a review aid, not a compliance product. Users must review its output.

## Why existing approaches are insufficient

- Cloud scanners expose sensitive content in transit and depend on connectivity.
- Pure pattern matching misses unstructured entities.
- Opaque acceleration claims make demos difficult to verify.
- Large generative models are unnecessary for this narrow classification task.

## Implementation status

| Capability | Status |
|---|---|
| Local browser UI and API | **IMPLEMENTED** |
| Structured PII recognition and Luhn validation | **IMPLEMENTED** |
| Lightweight contextual local classifier | **IMPLEMENTED — CPU fallback** |
| Backend/load/inference diagnostics | **IMPLEMENTED** |
| Network-free inference | **IMPLEMENTED for current feature** |
| QNN provider detection and safe fallback | **IMPLEMENTED** |
| Fine-tuned transformer ONNX artifact | **PLANNED / artifact not included** |
| AI Hub compile, physical-device profile, numerical validation | **PLANNED — requires account/hardware workflow** |
| Verified Snapdragon NPU execution | **NOT YET VERIFIED** |

## AI and Snapdragon architecture

```text
User → Local web UI → Local HTTP API → AIInferenceEngine
                                      ├─ QNN Execution Provider → Snapdragon NPU (when verified)
                                      └─ scikit-learn classifier → CPU fallback (current demo)
```

The target model is a task-fine-tuned DistilBERT-compatible token classifier exported to ONNX. Qualcomm AI Hub lists DistilBERT as applicable to named entity recognition and lists Snapdragon X Elite/X Plus devices. The current runnable fallback uses logistic regression over token/context features so every judge can demonstrate the complete workflow without credentials or a model download. It is not described as NPU optimized.

See [Qualcomm AI Hub setup](docs/QUALCOMM_AI_HUB.md), [profiling guide](docs/PROFILING_GUIDE.md), and [audit](docs/SNAPDRAGON_AUDIT.md).

## Requirements

- Python 3.10–3.12 (tested here with Python 3.11)
- Windows, macOS, or Linux for CPU demo
- Target deployment: Windows 11 on a Snapdragon X Elite/X Plus HP PC
- For planned NPU path: ARM64 ONNX Runtime with QNN Execution Provider and an AI Hub–compiled compatible model

## Install and run

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python -m snapmind.server
```

Open `http://127.0.0.1:8000`, select **Load sample**, then **Analyze locally**. The terminal and UI show the active backend.

Configuration is optional. Copy `.env.example` values into your shell environment; this minimal server intentionally does not auto-load `.env` files.

```powershell
$env:SNAPMIND_BACKEND = "cpu"   # auto | cpu | qnn
$env:SNAPMIND_MODEL_PATH = "models/pii_token_classifier.onnx"
python -m snapmind.server
```

## Tests

```powershell
python -m unittest discover -s tests -v
```

Tests cover normal inference, invalid/empty input, missing model, unavailable accelerator, structured identifiers, contextual detection, and the network-free contract. No API key is needed by the app; AI Hub credential setup is separately documented and deliberately untested here.

## Deployment and accessibility

The app binds to loopback by default, has keyboard-accessible controls, responsive layout, semantic labels, live result announcements, and high-contrast status text. For a shared LAN demo, explicitly pass `--host 0.0.0.0` only on a trusted network. Snapdragon deployment steps are in `docs/QUALCOMM_AI_HUB.md`.

## Performance

The UI reports actual per-request wall-clock inference time and startup model-load time. Numbers vary by device and are not hard-coded. Qualcomm AI Hub and Snapdragon hardware measurements are **not yet profiled**; no latency, power, TOPS, or speedup claim is made.

## Privacy and security

- Analysis is performed in the local Python process; the inference path makes no outbound request.
- Input is held in memory and is not persisted or logged by application code.
- The server binds to `127.0.0.1` by default and returns restrictive response headers.
- No secrets are required at runtime. AI Hub tokens belong in the user-level AI Hub configuration, never Git.
- If a user deliberately exposes the server to a network, submitted text can reach that host; TLS/authentication are not included in this demo.

## Limitations

- The small bundled training corpus is for demonstration and is not production-grade.
- Detection can miss or over-redact entities; English examples dominate.
- Pasted text only; PDF/OCR ingestion is not implemented.
- The accelerated ONNX model needs matching tokenization/post-processing work before deployment.
- NPU execution has not been verified in this environment.

## Roadmap

1. Fine-tune and evaluate a compact token classifier on licensed, de-identified NER/PII data.
2. Export fixed-shape ONNX, compile and profile through Qualcomm AI Hub on Snapdragon X Elite.
3. Add compatible local tokenizer/post-processing and parity tests.
4. Evaluate precision/recall by entity class and document a model card.
5. Package as a signed Windows ARM64 desktop application.

## Project structure

```text
snapmind/   inference layer and local server
web/           accessible demo UI
tests/         inference and fallback tests
models/        artifact placement instructions (binaries ignored)
docs/          audit, AI Hub guide, proposal, pitch sources
submission/    generated competition PDF/PPTX deliverables
scripts/       submission-file generator
```

## Screenshots and demo

Run the app, load the sample, and capture one screenshot showing the result plus execution status. A scripted two-minute demo is in `docs/DEMO_GUIDE.md`. Screenshots are a remaining manual step because no browser capture was required to validate inference.

## License and credits

Project code: MIT License. Python dependencies retain their own licenses. Planned model backbone: DistilBERT (Apache-2.0 per Qualcomm AI Hub listing). Qualcomm, Snapdragon, and Hexagon are trademarks of Qualcomm Incorporated or its subsidiaries; this is a challenge submission and does not imply endorsement.

