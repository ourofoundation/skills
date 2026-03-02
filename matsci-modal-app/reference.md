# Modal App Reference Template

Annotated template for building a new materials science Modal app with Ouro integration.

## Synchronous Pattern (jobs < 5 min)

Use when inference is fast enough to return directly.

```python
import base64
import os
from pathlib import Path
from typing import Optional
from uuid import uuid4

import modal
from fastapi import Header, HTTPException
from fastapi.openapi.utils import get_openapi
from ouro import Ouro
from ouro.models import Action
from ouro.utils import get_custom_openapi, ouro_field
from pydantic import BaseModel, Field

MINUTES = 60

# --- Volume for model weights and outputs ---
volume = modal.Volume.from_name("<model-name>-data", create_if_missing=True)
CONTAINER_PATH = Path("/<model-name>")
MODELS_PATH = CONTAINER_PATH / "models"
OUTPUT_PATH = CONTAINER_PATH / "output"

# --- Container image ---
image = (
    modal.Image.debian_slim(python_version="3.11")
    .apt_install(["git"])
    .pip_install(
        # ML framework
        "torch==2.2.1+cu118",
        extra_index_url="https://download.pytorch.org/whl/cu118",
    )
    .pip_install(
        # Model-specific deps
        "pymatgen>=2024.6.4",
        "ase>=3.22.1",
        "fastapi[standard]",
        "requests",
        "ouro-py",
        # Add model-specific packages here
    )
    .run_commands(
        # Clone model repo and install
        "git clone https://github.com/<org>/<model>.git /opt/<model>",
        "cd /opt/<model> && pip install -e . --no-deps",
    )
)

app = modal.App(
    name="<model-name>",
    image=image,
    secrets=[modal.Secret.from_name("mattergen")],
)


# --- Request/Response Models ---

class File(BaseModel):
    """Ouro file input model."""
    url: str
    filename: str
    name: Optional[str] = None
    description: Optional[str] = None
    id: Optional[str] = None
    type: str
    org_id: str
    team_id: str
    visibility: str


class PredictRequest(BaseModel):
    """Adjust fields to match the model's input."""
    file: File = Field(..., description="Input CIF file")
    # Add model-specific parameters as needed


# --- Modal function with FastAPI ---

@app.function(
    volumes={str(CONTAINER_PATH): volume},
    gpu="L4",
    timeout=10 * MINUTES,
)
@modal.asgi_app()
def serve():
    from fastapi import FastAPI

    web_app = FastAPI(
        title="<Model Name>",
        summary="<One-line summary>",
        description="<Longer description of the model>",
        version="1.0.0",
    )
    web_app.openapi = get_custom_openapi(web_app, get_openapi)

    @web_app.post(
        "/<model-name>/predict",
        summary="<Endpoint summary>",
        description="<Endpoint description>",
    )
    @ouro_field("x-ouro-input-asset-type", "file")
    @ouro_field("x-ouro-input-file-extension", "cif")
    @ouro_field("x-ouro-output-asset-type", "file")
    @ouro_field("x-ouro-output-file-extension", "cif")
    async def predict(
        request: PredictRequest,
        ouro_route_id: Optional[str] = Header(None, alias="ouro-route-id"),
        ouro_route_org_id: Optional[str] = Header(None, alias="ouro-route-org-id"),
        ouro_route_team_id: Optional[str] = Header(None, alias="ouro-route-team-id"),
        ouro_action_id: Optional[str] = Header(None, alias="ouro-action-id"),
    ):
        import requests as req

        ouro = Ouro(
            api_key=os.environ["OURO_API_KEY"],
            base_url=os.environ["OURO_BACKEND_URL"],
            database_url=os.environ["SUPABASE_URL"],
        )

        action = ouro.routes.retrieve_action(ouro_action_id) if ouro_action_id else None

        if action:
            action.log("Starting prediction...")

        # Download input file
        file_url = request.file.url
        supabase_url = os.getenv("SUPABASE_URL")
        file_url = file_url.replace("http://localhost:54321", supabase_url)
        file_url = file_url.replace("http://127.0.0.1:54321", supabase_url)
        response = req.get(file_url, timeout=30)
        response.raise_for_status()
        input_content = response.text

        # === MODEL INFERENCE GOES HERE ===
        # result = model.predict(input_content)
        # output_bytes = result.to_cif()

        # Encode output
        output_b64 = base64.b64encode(output_bytes).decode("utf-8")

        if action:
            action.log("Prediction complete")

        return {
            "file": {
                "name": "Prediction result",
                "description": "Output from <model-name>",
                "filename": "result.cif",
                "type": "text/cif",
                "extension": "cif",
                "base64": output_b64,
                "org_id": ouro_route_org_id,
                "team_id": ouro_route_team_id,
            }
        }

    return web_app
```

## Asynchronous Webhook Pattern (long-running jobs)

Use when compute takes > 5 min. Webapp returns 202 immediately, spawns compute, sends result via webhook.

```python
import json
import os
import time
import uuid
from pathlib import Path
from typing import Optional

import modal
import requests
from fastapi import FastAPI, Header, HTTPException, BackgroundTasks, status
from fastapi.responses import JSONResponse
from fastapi.openapi.utils import get_openapi
from ouro import Ouro
from ouro.models import Action
from ouro.utils import get_custom_openapi, ouro_field
from pydantic import BaseModel, Field

app = modal.App(
    "<model-name>-calculator",
    secrets=[modal.Secret.from_name("mattergen")],
)

volume = modal.Volume.from_name("<model-name>-data", create_if_missing=True)

# Lightweight webapp image (fast cold starts)
webapp_image = (
    modal.Image.debian_slim(python_version="3.12")
    .pip_install(
        "fastapi>=0.104.0",
        "requests",
        "ouro-py",
        "pymatgen>=2023.10.0",
    )
)

# Heavy compute image (with CUDA, model deps)
compute_image = (
    modal.Image.from_registry("nvidia/cuda:12.9.1-devel-ubuntu24.04", add_python="3.12")
    .pip_install(
        # All heavy dependencies here
        "ouro-py",
    )
    .run_commands(
        # Clone and install model
    )
)


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


class ComputeRequest(BaseModel):
    file: File = Field(..., description="Input file")


def _normalize_webhook_url(url: str) -> str:
    ouro_backend_url = os.getenv("OURO_BACKEND_URL", "")
    url = url.replace("http://localhost:8003", ouro_backend_url)
    url = url.replace("http://127.0.0.1:8003", ouro_backend_url)
    return url


def _send_webhook_notification(
    ouro_webhook_url: Optional[str],
    ouro_webhook_token: Optional[str],
    ouro_action_id: Optional[str],
    ouro_route_id: Optional[str],
    status: str,
    response: dict,
):
    if not ouro_webhook_url:
        return
    ouro_webhook_url = _normalize_webhook_url(ouro_webhook_url)
    payload = {
        "status": status,
        "ouro_action_id": ouro_action_id,
        "ouro_route_id": ouro_route_id,
        "response": response,
    }
    headers = {"Content-Type": "application/json"}
    if ouro_webhook_token:
        headers["ouro-webhook-token"] = ouro_webhook_token
    try:
        requests.post(ouro_webhook_url, json=payload, headers=headers, timeout=30)
    except Exception as e:
        print(f"Webhook error: {e}")


# --- Lightweight webapp (fast cold start) ---

@app.cls(image=webapp_image)
class WebApp:
    web_app = FastAPI(
        title="<Model Name>",
        summary="<Summary>",
        version="1.0.0",
    )

    @modal.enter()
    def setup(self):
        self.setup_routes()
        self.web_app.openapi = get_custom_openapi(self.web_app, get_openapi)

    def setup_routes(self):

        @ouro_field("x-ouro-input-asset-type", "file")
        @ouro_field("x-ouro-input-file-extension", "cif")
        @self.web_app.post(
            "/<model>/compute",
            summary="<Summary>",
        )
        async def compute(
            request: ComputeRequest,
            ouro_route_id: Optional[str] = Header(None, alias="ouro-route-id"),
            ouro_action_id: Optional[str] = Header(None, alias="ouro-action-id"),
            ouro_webhook_url: Optional[str] = Header(None, alias="ouro-webhook-url"),
            ouro_webhook_token: Optional[str] = Header(None, alias="ouro-webhook-token"),
        ):
            if not ouro_webhook_url:
                raise HTTPException(status_code=400, detail="Missing ouro-webhook-url header")

            ouro_webhook_url = _normalize_webhook_url(ouro_webhook_url)

            # Download and validate input
            file_url = request.file.url
            supabase_url = os.getenv("SUPABASE_URL")
            file_url = file_url.replace("http://localhost:54321", supabase_url)
            file_url = file_url.replace("http://127.0.0.1:54321", supabase_url)
            resp = requests.get(file_url, timeout=30)
            resp.raise_for_status()
            content = resp.text

            # Spawn heavy compute in separate container
            modal_request = {
                "content": content,
                "ouro_action_id": ouro_action_id,
                "ouro_route_id": ouro_route_id,
                "ouro_webhook_url": ouro_webhook_url,
                "ouro_webhook_token": ouro_webhook_token,
            }
            HeavyCompute().run_with_webhook.spawn(modal_request)

            return JSONResponse(
                status_code=status.HTTP_202_ACCEPTED,
                content={
                    "status": "accepted",
                    "message": "Computation started. You will be notified when complete.",
                },
            )

    @modal.asgi_app(label="<model>-api")
    def serve(self):
        return self.web_app


# --- Heavy compute (GPU, long timeout) ---

@app.cls(
    image=compute_image,
    gpu="A100",
    volumes={"/data": volume},
    timeout=3600 * 2,
)
class HeavyCompute:

    @modal.enter()
    def setup(self):
        self.result = None
        self.ouro_webhook_url = None
        self.ouro_webhook_token = None
        self.ouro_action_id = None
        self.ouro_route_id = None
        self.webhook_sent = False
        self.calculation_started = False

    @modal.exit()
    def cleanup(self):
        """Send webhook on exit if calculation completed but webhook wasn't sent."""
        if self.webhook_sent or not self.ouro_webhook_url or not self.calculation_started:
            return
        if not self.result:
            return
        status_str = "completed" if self.result.get("status") != "error" else "failed"
        _send_webhook_notification(
            self.ouro_webhook_url, self.ouro_webhook_token,
            self.ouro_action_id, self.ouro_route_id,
            status_str, self.result,
        )
        self.webhook_sent = True

    @modal.method()
    def run_with_webhook(self, request: dict):
        self.ouro_webhook_url = request.get("ouro_webhook_url")
        self.ouro_webhook_token = request.get("ouro_webhook_token")
        self.ouro_action_id = request.get("ouro_action_id")
        self.ouro_route_id = request.get("ouro_route_id")

        try:
            self.calculation_started = True

            # Retrieve the Action object for structured logging
            action = None
            if self.ouro_action_id:
                ouro = Ouro(
                    api_key=os.environ["OURO_API_KEY"],
                    base_url=os.getenv("OURO_BACKEND_URL"),
                    database_url=os.getenv("SUPABASE_URL"),
                )
                action = ouro.routes.retrieve_action(self.ouro_action_id)

            if action:
                action.log("Starting computation...")

            # === RUN COMPUTATION ===
            # self.result = run_model(request.get("content"), action=action, ...)

            if action:
                action.log("Computation complete")

            status_str = "completed" if self.result.get("status") != "error" else "failed"
            _send_webhook_notification(
                self.ouro_webhook_url, self.ouro_webhook_token,
                self.ouro_action_id, self.ouro_route_id,
                status_str, self.result,
            )
            self.webhook_sent = True
            return self.result

        except Exception:
            import traceback
            self.result = {"status": "error", "error": traceback.format_exc()}
            if action:
                action.log("Computation failed", level="error")
            _send_webhook_notification(
                self.ouro_webhook_url, self.ouro_webhook_token,
                self.ouro_action_id, self.ouro_route_id,
                "failed", self.result,
            )
            self.webhook_sent = True
            return self.result
```

## Common File Extensions in Materials Science

| Extension | MIME Type | Description |
|-----------|-----------|-------------|
| `.cif` | `text/cif` or `application/cif` | Crystallographic Information File |
| `.poscar` / `.vasp` | `text/plain` | VASP structure format |
| `.xyz` / `.extxyz` | `text/plain` | Atomic coordinates |
| `.json` | `application/json` | Structured results |
| `.zip` | `application/zip` | Multiple output files |
| `.html` | `text/html` | Interactive visualizations |
| `.csv` | `text/csv` | Tabular data |
| `.traj` | `application/octet-stream` | ASE trajectory |

## Modal Deployment Commands

```bash
# Deploy app (creates/updates the Modal deployment)
modal deploy materials/apps/<model-name>/app.py

# Run local entrypoint for testing
modal run materials/apps/<model-name>/app.py --arg value

# View logs
modal app logs <app-name>

# List running apps
modal app list
```
