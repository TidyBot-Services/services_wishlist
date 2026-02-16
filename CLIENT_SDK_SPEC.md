# Client SDK Specification

## Background

Each service in the catalog provides a `client.py` that frontend (skill) agents use to call the service from the robot. The `client.py` is hosted in the service's GitHub repo and auto-synced to the robot via `sync_catalog.sh`.

**How frontend agents use services:**

```
Frontend agent writes code → POST /code/execute → runs in sandbox → imports client.py → calls backend service
```

The code execution sandbox on the robot has restrictions. Client SDKs **must** follow this spec to work inside the sandbox.

---

## Requirements

### 1. Use `urllib` only (not `requests`)

The sandbox blocks `requests`, `http.client`, and `httpx`. Only `urllib.request` and `urllib.error` are available.

**Do this:**
```python
import urllib.request
import urllib.error
import json

data = json.dumps(payload).encode("utf-8")
req = urllib.request.Request(url, data=data, headers={"Content-Type": "application/json"})
with urllib.request.urlopen(req, timeout=self.timeout) as resp:
    return json.loads(resp.read().decode("utf-8"))
```

**Not this:**
```python
import requests  # BLOCKED in sandbox
r = requests.post(url, json=payload)
```

### 2. Allowed imports

Safe imports available in the sandbox:

```python
import urllib.request, urllib.error, urllib.parse  # HTTP
import json, base64, io, os, time, math            # stdlib
import numpy as np                                  # arrays
import cv2                                          # image encoding/decoding (optional)
```

Do NOT rely on: `requests`, `httpx`, `http.client`, `aiohttp`, `grpc`.

### 3. Accept image data as bytes

Frontend agents get camera frames as raw bytes from the robot SDK (`sensors.get_camera_frame()`). The client must accept `bytes` as input — not just file paths.

```python
def detect(self, image, ...):
    """
    Args:
        image: Image as bytes (JPEG/PNG), file path (str), numpy array (H,W,3), or base64 string.
    """
```

### 4. Constructor takes `host` parameter

The host URL must be configurable. Use the service's actual backend URL as the default:

```python
class MyServiceClient:
    def __init__(self, host: str = "http://<backend-ip>:<port>", timeout: float = 60.0):
        self.host = host.rstrip("/")
        self.timeout = timeout
```

### 5. Include a `health()` method

Every client must have a `health()` method so agents can check if the service is up:

```python
def health(self) -> dict:
    req = urllib.request.Request(f"{self.host}/health")
    with urllib.request.urlopen(req, timeout=self.timeout) as resp:
        return json.loads(resp.read().decode("utf-8"))
```

### 6. Module-level docstring with usage examples

The docstring is the primary documentation for frontend agents. Include concrete examples showing how to use the client inside the robot's code execution sandbox:

```python
"""
<Service Name> — Python Client SDK

Usage (inside robot code execution):
    from services.<service_name>.client import <ClientClass>
    from robot_sdk import sensors

    client = <ClientClass>()

    # Get camera frame from robot
    frame_bytes = sensors.get_camera_frame()

    # Call the service
    result = client.<method>(frame_bytes, ...)
    for item in result:
        print(item)
"""
```

---

## Template

Below is a minimal complete `client.py` template. Copy and adapt for your service:

```python
"""
<Service Name> — Python Client SDK

Usage (inside robot code execution):
    from services.<service_name>.client import <ServiceName>Client
    from robot_sdk import sensors

    client = <ServiceName>Client()

    # Get camera frame from robot
    frame_bytes = sensors.get_camera_frame()

    # Call the service
    result = client.<main_method>(frame_bytes, ...)
"""

import base64
import json
import urllib.request
import urllib.error
from pathlib import Path
from typing import Optional

import numpy as np


class <ServiceName>Client:
    """Client SDK for the <Service Name> service."""

    def __init__(self, host: str = "http://<backend-ip>:<port>", timeout: float = 60.0):
        self.host = host.rstrip("/")
        self.timeout = timeout

    def health(self) -> dict:
        """Check if the service is running."""
        req = urllib.request.Request(f"{self.host}/health")
        with urllib.request.urlopen(req, timeout=self.timeout) as resp:
            return json.loads(resp.read().decode("utf-8"))

    def _encode_image(self, image) -> str:
        """Encode image to base64 from bytes, file path, numpy array, or pass through."""
        if isinstance(image, bytes):
            return base64.b64encode(image).decode()
        elif isinstance(image, (str, Path)):
            return base64.b64encode(Path(image).read_bytes()).decode()
        elif isinstance(image, np.ndarray):
            import cv2
            _, buf = cv2.imencode(".png", image)
            return base64.b64encode(buf.tobytes()).decode()
        return image  # assume already base64

    def _post(self, endpoint: str, payload: dict) -> dict:
        """POST JSON to service and return parsed response."""
        data = json.dumps(payload).encode("utf-8")
        req = urllib.request.Request(
            f"{self.host}{endpoint}",
            data=data,
            headers={"Content-Type": "application/json"},
        )
        try:
            with urllib.request.urlopen(req, timeout=self.timeout) as resp:
                return json.loads(resp.read().decode("utf-8"))
        except urllib.error.HTTPError as e:
            body = e.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"{endpoint} failed (HTTP {e.code}): {body}") from e
        except urllib.error.URLError as e:
            raise RuntimeError(f"Service unavailable at {self.host}: {e.reason}") from e

    # -- Add service-specific methods below --

    # def detect(self, image, ...) -> list[dict]:
    #     payload = {"image": self._encode_image(image), ...}
    #     return self._post("/detect", payload)["detections"]
```

---

## Catalog Documentation

When adding a service to `catalog.json`, include enough information for a frontend agent to use the service without reading source code.

### Required fields

```json
{
  "type": "model|service",
  "description": "What the service does (1-2 sentences)",
  "host": "http://<backend-ip>:<port>",
  "endpoints": ["GET /health", "POST /detect"],
  "client_sdk": "https://raw.githubusercontent.com/<org>/<repo>/<branch>/client.py",
  "service_repo": "https://github.com/<org>/<repo>",
  "api_docs": "https://github.com/<org>/<repo>/blob/<branch>/README.md",
  "version": "0.1.0",
  "added_by": "agent-name",
  "added_at": "YYYY-MM-DD"
}
```

### Required: add `usage` field

Add a `usage` field showing how a frontend agent imports and calls the client inside the code execution sandbox. This is the single most useful piece of documentation for skill agents:

```json
{
  "usage": {
    "import": "from services.<service_name>.client import <ClientClass>",
    "init": "client = <ClientClass>()",
    "example": "result = client.<method>(image_bytes, ...)",
    "returns": "Description of what the method returns"
  }
}
```

---

## File Layout on Robot

After `sync_catalog.sh` runs, each service ends up at:

```
tidybot-agent-server/
  services/
    <service_name>/
      client.py       # Downloaded from client_sdk URL
      service.json    # Auto-generated metadata from catalog
```

Frontend agents import as: `from services.<service_name>.client import <ClientClass>`

---

## Checklist

When creating or updating a service's `client.py`:

- [ ] Uses `urllib.request` (not `requests`)
- [ ] Only uses allowed imports (see section 2)
- [ ] Accepts image input as `bytes`, file path, numpy array, or base64
- [ ] Constructor has `host` parameter with correct default URL
- [ ] Has `health()` method
- [ ] Module docstring shows usage with `from services.<service_name>.client import ...`
- [ ] Catalog entry has `usage` field with import/init/example/returns
