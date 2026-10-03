# Open-Source Air-Gapped Document Data Automation — Initial Draft

Sep 29, 2026 · @David

## Overview

This project is an open-source, fully air-gapped alternative to Bedrock Data Automation (BDA). It turns PDFs into schema-bound JSON with per-field bounding boxes and calibrated confidence scores, using only open-weight models.

The problem: managed services like BDA and Azure Document Intelligence are unavailable or hard to accredit in disconnected, classified, or IL5+ environments. Existing open-source libraries usually depend on hosted LLMs or cloud OCR for grounding, and report raw LLM confidence, which is poorly calibrated.

Working name: TBD. The target users are teams in regulated environments (defense, govtech, healthcare, finance) who need BDA-style automation without data leaving their boundary.

## Goals and non-goals

Goals:

- **Air-gapped end to end.** No network calls at runtime; all models are open-weight and served locally.
- **BDA-style blueprints in.** Users define extraction targets in a JSON format modeled on BDA custom blueprints.
- **BDA-style JSON out.** Results mirror BDA's custom output shape, including per-field geometry and confidence.
- **Calibrated confidence.** Scores that mean what they say (a 0.9 is right about 90% of the time), not raw logprobs.
- **Built-in evaluation harness.** Accuracy, grounding, and calibration metrics against labeled data, runnable in CI.
- **Scales horizontally.** Batch processing of large PDF volumes on Kubernetes with GPU workers.

Non-goals (for v1):

- Wire-level API compatibility with BDA (similar formats, not a drop-in endpoint).
- Audio, video, and image-only modalities.
- A hosted SaaS offering.
- Training new foundation models; fine-tuning is optional, not required.

## Blueprint input format

Blueprints follow BDA's custom-blueprint style: a JSON Schema document with a `class`, a `description`, and per-field `instruction` and `inferenceType`. Field names and structure are modeled on BDA from memory and should be verified against current AWS docs before we lock them down.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "class": "Invoice",
  "description": "Commercial invoice from a vendor",
  "type": "object",
  "properties": {
    "invoice_number": {
      "type": "string",
      "inferenceType": "explicit",
      "instruction": "The invoice ID printed near the top"
    },
    "total_due": {
      "type": "number",
      "inferenceType": "explicit",
      "instruction": "Final amount due including tax"
    },
    "is_overdue": {
      "type": "boolean",
      "inferenceType": "inferred",
      "instruction": "True if the due date is before the processing date"
    }
  }
}
```

Key behaviors:

- **explicit** fields must be grounded to text on the page, so they get geometry.
- **inferred** fields are derived by the model; they cite supporting evidence but may have no single bounding box.
- Blueprints compile to Pydantic models internally, which drive constrained decoding and validation.
- Optional project extensions (namespaced, e.g. `x-validators`) add deterministic checks like regex, ranges, and cross-field sums.
- Multiple blueprints per project enable document classification: the pipeline picks the best-matching blueprint first.

## Output format

Output mirrors BDA's custom-output shape: the matched blueprint, the inference result, and an explainability block with per-field confidence and geometry. As with blueprints, verify exact BDA field names before finalizing.

```json
{
  "matched_blueprint": { "name": "Invoice", "confidence": 0.97 },
  "document_class": { "type": "Invoice" },
  "inference_result": {
    "invoice_number": "INV-20417",
    "total_due": 1842.50,
    "is_overdue": false
  },
  "explainability_info": [
    {
      "invoice_number": {
        "success": true,
        "confidence": 0.96,
        "value": "INV-20417",
        "type": "string",
        "geometry": [
          { "page": 1, "boundingBox": { "left": 0.62, "top": 0.08, "width": 0.14, "height": 0.02 } }
        ]
      }
    }
  ],
  "x-provenance": { "models": {}, "pipeline_version": "0.1.0" }
}
```

Notes:

- Bounding boxes are page-normalized (0 to 1), matching BDA convention.
- `confidence` is the calibrated score, not a raw model probability.
- `x-` prefixed fields are project extensions (provenance, raw signal breakdown, review flags) that BDA consumers can ignore.
- A standard-output mode (full page text, layout elements, tables with geometry) is a later addition.

## Architecture

The pipeline runs in seven stages inside the air-gapped boundary, with a review-and-evaluation loop that feeds corrections back into calibration.

&#91;embedded content: pipeline architecture · 7 stages plus review and eval loop\]

Layout/OCR and LLM extraction run as separate GPU worker pools, because their throughput profiles differ. Low-confidence fields go to the review UI; corrections become labeled data that retrains the calibrator and feeds the eval harness.

## Agentic repair loop

The seven-stage pipeline is the fast path; an agent runs only on fields that fail validation or fall below the confidence threshold. This keeps the cost of agency proportional to document difficulty, and it gives the eval harness a clean comparison: fast path alone vs. fast path plus repair.

The agent controls the path, not the tools. On each step it reads the draft output, the validator report, and the calibrated scores, then picks one action: re-run a tool on a region, search other pages for evidence, choose between conflicting candidates, accept the field, or abstain. Deterministic code executes the action and re-validates. Two documents of the same type can take different paths, and every path is recorded as a trace.

The agent makes four kinds of decisions:

- **Repair strategy.** A failed field is retried a different way: re-OCR at higher DPI, crop the region and ask the VLM, or switch to table extraction.
- **Evidence seeking.** A missing field triggers a search of other pages, including references such as "see Attachment B" and continuation pages.
- **Conflict resolution.** When two pages give different values, the agent picks one and records why.
- **Abstention.** When repairs are exhausted, the agent returns the field flagged for the review UI rather than guessing.

&#91;embedded content: agentic repair loop · validate, decide, act, re-validate\]

A failing field goes to the agent, which picks one tool action; the result is re-validated, and the loop ends when the field passes or the agent abstains.

### Agent tools

Every tool is read-only and returns word IDs or boxes, so any value the agent proposes is grounded the same way as fast-path output.

| Tool | What it does | Returns |
| --- | --- | --- |
| `ocr(page, region, dpi)` | Re-runs docTR on a region at a chosen resolution | Words, boxes, OCR confidence |
| `extract_table(page, region)` | Runs Docling table extraction on a region | Rows, cells, boxes |
| `vlm_read(page, region, question)` | Asks the VLM a targeted question about a cropped region | Answer, cited word IDs |
| `search_pages(query)` | Finds candidate pages and regions for a field across the document | Ranked page and region list |
| `validate(draft, blueprint)` | Runs schema, regex, and cross-field checks | Errors, low-confidence fields |
| `accept(field)` / `abstain(field, reason)` | Ends work on a field | Final status |

### Guardrails

- **Step budget.** At most 12 actions per document and 3 per field; exhausting either flags the remaining fields as `budget_exhausted`.
- **Provenance required.** A value without word IDs from a tool result is rejected, so the agent cannot assert values from its own text.
- **Deterministic checks decide.** The validator and calibrated scorer, not the agent, decide whether a field passes.
- **Air-gapped.** The agent model runs on the same vLLM pool; no tool makes a network call.

### How the eval harness measures it

The harness runs every labeled document twice, fast path only and fast path plus repair, and reports:

- Field accuracy and ECE for both runs, and the share of fast-path failures the loop recovers
- Abstention precision: how often an abstained field was actually wrong or missing on the fast path
- Cost per document: agent steps, GPU seconds, and p95 latency added by the loop
- Traces for each repaired field, viewable in the review UI

## Calibrated confidence

Confidence is a small learned model over several signals, calibrated against labeled data. This is the project's main differentiator.

Input signals per field:

- **OCR quality:** mean and minimum word confidence inside the field's box, plus region image sharpness.
- **Grounding:** whether the value literally or fuzzily matches the cited text; the fraction of cited IDs that exist.
- **Model signals:** token logprobs for the value; agreement across two prompts or two models (optional, costs extra compute).
- **Validators:** schema, type, regex, range, and cross-field checks passed or failed.
- **Field metadata:** explicit vs inferred, field type, blueprint.

Calibration approach:

1. Start with a hand-weighted heuristic so v0 works with no labeled data.
2. Once labels exist, fit a gradient-boosted classifier (P(field correct)) on the signals.
3. Apply isotonic regression or temperature scaling per field type, and report expected calibration error (ECE).
4. Ship calibrators as versioned artifacts; users can retrain on their own documents inside their boundary.

The output confidence then drives routing: auto-accept above a threshold, human review below it.

## Evaluation harness

The harness is a first-class CLI command (`eval`) that scores any pipeline configuration against labeled documents and produces a report. It exists from day one, so every model or prompt change is measured.

| Metric | What it measures | Notes |
| --- | --- | --- |
| Field accuracy | Exact and normalized match per field | Normalizers for dates, currency, whitespace |
| Precision / recall | Missing vs hallucinated fields | Nulls scored explicitly |
| Grounding IoU | Overlap of predicted vs labeled boxes | Only for explicit fields |
| ECE and reliability curve | Calibration quality of confidence | Per field type and overall |
| Review-rate tradeoff | Accuracy of auto-accepted fields at each threshold | The number ops teams care about |
| Throughput and cost | Pages per GPU-hour, latency p50/p95 | Per pipeline stage |

Components:

- **Dataset format:** a PDF plus a ground-truth JSON in the same BDA-style output shape, so labels and predictions diff directly.
- **Public benchmarks:** adapters for FUNSD, CORD, and similar datasets for reproducible published numbers.
- **Review UI:** a lightweight local web app showing PDF pages with box overlays, used both to label ground truth and to review low-confidence fields. Corrections feed back into the calibration dataset.
- **Regression gating:** a CI job fails if accuracy or ECE regresses beyond a set tolerance.

## Deployment, tech stack, and licensing

The project is Python-first with a permissive license, and deploys two ways: a single-node Docker Compose for evaluation, and Helm charts for Kubernetes at scale.

| Layer | Initial choice | Alternatives |
| --- | --- | --- |
| API and SDK | FastAPI, Pydantic v2, Python SDK + CLI | — |
| Layout and parsing | Docling (MIT) | PaddleOCR PP-Structure (Apache 2.0) |
| Word-level OCR | docTR (Apache 2.0) | PaddleOCR |
| LLM serving | vLLM with structured outputs and logprobs | SGLang |
| Default models | Qwen-class open-weight LLM and VLM | Llama, Mistral, Granite |
| Orchestration | Ray (or Celery) with a job queue | Temporal |
| Storage | S3-compatible (MinIO) + Postgres for jobs and results | — |
| Calibration | scikit-learn / LightGBM | — |
| Deployment | Helm on Kubernetes, GPU nodes autoscaled (e.g. Karpenter) | Docker Compose single node |

Air-gap requirements:

- Model weights and container images are bundled for offline transfer, with checksums and SBOMs.
- No telemetry, no runtime downloads; a startup check fails if anything tries to reach the network.
- Image signing and SBOM generation in CI (e.g. cosign, Syft) to support ATO and supply chain reviews.

Licensing: Apache 2.0 for the project. Every default dependency and model must have a permissive license; anything with commercial restrictions (for example, some OCR model weights) is an optional plugin only.

## Open questions and next steps

Open questions:

- Text-LLM over OCR output, or a VLM reading page images directly, as the default extraction path? (Could support both; which is primary?)
- How closely should blueprints and output track BDA? Exact field names, or "BDA-inspired" with a converter?
- Minimum labeled documents needed per blueprint before a learned calibrator beats the heuristic?
- Ray vs Celery for orchestration at v1?
- Project name and governance (solo, foundation, or company-backed)?

Next steps:

- [ ] Verify current BDA blueprint and output schemas against AWS documentation
- [ ] Build a thin end-to-end slice: one blueprint, Docling + vLLM, BDA-style output with boxes
- [ ] Stand up the eval harness on one public dataset (e.g. CORD) and publish baseline numbers
- [ ] Implement the heuristic confidence score, then the first learned calibrator
- [ ] Confirm no overlap with current-employer IP before publishing
