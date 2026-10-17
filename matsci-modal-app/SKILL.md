---
name: matsci-modal-app
description: Build Modal apps for open-source materials science AI models with Ouro integration. Use when the user wants to deploy a new AI model as a Modal app, find recent open-source AI-for-science models, or create a new app under materials/apps/ with file I/O and webhook support.
---

# Materials Science Modal App Builder

Build and deploy open-source materials science AI models as Modal apps integrated with Ouro.

## Workflow

### Phase 1: Discover a Model

Search the web for recent open-source AI models in materials science. Focus on:

- **Crystal structure prediction/generation** (diffusion, GNN, transformer)
- **Property prediction** (formation energy, band gap, elastic moduli, magnetism)
- **Molecular dynamics / force fields** (universal potentials, MLFF)
- **Characterization** (XRD, electron density, spectroscopy)
- **Inverse design** (conditional generation, optimization)

Prioritize models that have:

1. Published source code on GitHub
2. Released pretrained model weights (HuggingFace, Zenodo, or repo)
3. A clear inference API or CLI
4. A recent paper (last ~2 years)

Good search queries:

- `"materials science" AI model open source 2025 2026 github`
- `crystal structure generation model weights released`
- `universal machine learning force field open source`
- `materials property prediction pretrained model`
- `site:github.com materials science AI model`
- `site:huggingface.co materials science`

Present findings to the user with: model name, what it does, paper link, repo link, license, and input/output format.

### Phase 2: Understand the Model

Once the user picks a model:

1. Clone or read the repo's README, installation instructions, and inference examples
2. Identify:
   - **Input format**: CIF, POSCAR, SMILES, composition string, XRD pattern, etc.
   - **Output format**: CIF, JSON properties, trajectories, images, etc.
   - **Dependencies**: PyTorch/JAX version, CUDA requirements, special packages
   - **Model weights**: Where to download, how to load
   - **Inference API**: Python function signature or CLI command

### Phase 3: Build the Modal App

Create a new directory under `materials/apps/<model-name>/` following established patterns.

#### File Structure

```
materials/apps/<model-name>/
├── app.py              # Modal app with FastAPI endpoints
└── README.md           # Brief description and deployment notes
```

#### Modal App Template

See [reference.md](reference.md) for the complete annotated template.

Key decisions:

- **Sync vs Async**: Use `@modal.asgi_app()` with direct return for jobs < 5 min. Use webhook pattern (return 202, spawn compute, POST result) for longer jobs.
- **GPU**: Choose based on model size — `T4` for small models, `L4` for medium, `A100` for large
- **Image**: Start with `modal.Image.debian_slim()` for CPU-only or `modal.Image.from_registry("nvidia/cuda:...")` for GPU

#### Ouro Integration Checklist

- [ ] Accept Ouro headers: `ouro-route-id`, `ouro-action-id`, `ouro-route-org-id`, `ouro-route-team-id`
- [ ] For async jobs: also accept `ouro-webhook-url`, `ouro-webhook-token`
- [ ] If the route is priced per second: accept `ouro-max-billable-seconds` (the run's billing budget) and stop or reject jobs that can't finish within it; optionally report the run's own billable time, as `usage.seconds` in the completion webhook or, for a sync route, in an `ouro-billable-seconds` response header (it can only lower the charge)
- [ ] Decorate endpoints with `@ouro_field("x-ouro-output-assets", {"file": {"asset_type": "file", "primary": True, "file_extensions": ["cif"]}})`
- [ ] If accepting file input: `@ouro_field("x-ouro-input-assets", {"file": {"asset_type": "file", "primary": True, "file_extensions": ["cif"]}})`, keyed by the request body field that receives the file
- [ ] If the response returns a list of records (a curve, a ranking, a phase list): declare `@ouro_field("x-ouro-response-charts", [{"data": "<list key>", "title": "...", "sort": "-score", "config": {...}}])` so Ouro charts every run from the JSON. `config` is a dataset view chart config keyed by the records' own keys; `{path}` / `{path:0}` in strings and `{"field": "path"}` as a reference line value read per-run values. Keep the list inline. For a dataset copy, return it under its own key (never the inline key) behind `include_dataset: bool = False`, declare it `{"asset_type": "dataset", "when": {"include_dataset": True}}`, and set the chart's `"dataset"` to that key so the chart becomes its view. See https://ouro.foundation/docs/concepts/routes#charts-on-a-run
- [ ] Use `get_custom_openapi(web_app, get_openapi)` for OpenAPI schema
- [ ] Return file output as base64 with Ouro file metadata
- [ ] Don't create a post per run by default. Public posts are announced in the team feed, and the action already links a run's output files. If a write-up is useful, put it behind a request flag (e.g. `include_report: bool = False`) and declare the post output `"when": {"include_report": True}` (files that go into it get `"embedded_in": "report"`). Use `when` for any output a request field switches on; `"optional": True` only means the results may not produce it
- [ ] For async: send webhook with status, action_id, route_id, and response
- [ ] Log progress via `Action` model: `action.log("message")`, `action.log("error", level="error")`
- [ ] Add `"ouro-py"` to image pip_install
- [ ] Use `modal.Secret.from_name("mattergen")` for env vars (OURO_API_KEY, OURO_BACKEND_URL, SUPABASE_URL)

#### Action Logging Pattern

Use the `Action` model from `ouro.models` for structured logging instead of raw HTTP calls:

```python
from ouro.models import Action

# In your endpoint, retrieve the action object once
action = ouro.routes.retrieve_action(ouro_action_id) if ouro_action_id else None

# Then pass `action` to any function that needs to log
if action:
    action.log("Starting prediction...")
    action.log("Generation progress: 50%")
    action.log("Generation failed", level="error")
    action.log("Fallback used", level="warning")
```

The `Action.log()` method automatically resolves the route_id and posts to the correct endpoint. Pass `action: Optional[Action]` through your call chain instead of separate `ouro_client`/`action_id`/`route_id` parameters.

#### File Input Pattern

For endpoints that accept Ouro files:

```python
class File(BaseModel):
    url: str
    filename: str
    name: Optional[str] = None
    description: Optional[str] = None
    id: Optional[str] = None
    type: str
    org_id: str
    team_id: str
    visibility: str
```

Download the file from `file.url`, normalizing localhost URLs for prod:

```python
supabase_url = os.getenv("SUPABASE_URL")
file_url = file_url.replace("http://localhost:54321", supabase_url)
file_url = file_url.replace("http://127.0.0.1:54321", supabase_url)
```

#### File Output Pattern

Return base64-encoded file with metadata:

```python
return {
    "file": {
        "name": "Human-readable name",
        "description": "What this file contains",
        "filename": "output.cif",
        "type": "text/cif",          # MIME type
        "extension": "cif",           # File extension
        "base64": base64_encoded_str,
        "org_id": ouro_route_org_id,
        "team_id": ouro_route_team_id,
    }
}
```

Common MIME types: `text/cif`, `application/zip`, `text/csv`, `application/json`, `text/html`

### Phase 4: Deploy and Test

```bash
# Deploy to Modal
modal deploy materials/apps/<model-name>/app.py

# Test locally first
modal run materials/apps/<model-name>/app.py
```

## Existing Apps for Reference

Look at these in `materials/apps/` as working examples:

| App                                  | Pattern                   | Key Features                                                     |
| ------------------------------------ | ------------------------- | ---------------------------------------------------------------- |
| `mattergen/app.py`                   | Sync ASGI + GPU           | Generation from text params, zip/cif output, symmetry correction |
| `mcae-predictor/abacus/modal_app.py` | Async webhook + spawn     | CIF file input, long compute, webhook callback, progress logging |
| `electron-density/modal_app.py`      | Standalone function       | Simple CIF input, volume storage, no FastAPI                     |
| `materials/app.py`                   | Webapp + separate compute | Routes that spawn remote Modal functions                         |
