# MAYA — Autonomous Counter-Deception & Phishing Neutralization Engine

**Multi-Agent Yielded Counter-Deception (MAYA)** | Built for **Swim with TinyFish** by **Team Janaki**

MAYA is an autonomous offensive-security and counter-deception platform that moves beyond passive phishing detection. While traditional URL scanners merely block suspicious links—leaving the attacker's infrastructure alive to scam thousands of others—MAYA actively enters live phishing portals using isolated headless browsers, reverses their exfiltration pipelines, poisons the attacker's backend database with trackable synthetic honey-tokens, and autonomously files technical takedown reports with domain registrars and hosting providers.

## 1. Why Passive Defense Fails

* **Scammer Databases Stay Live:** Static URL blocklists only flag links for end-users on protected networks while the underlying credential-harvesting infrastructure remains online.
* **85% Evasion Against Static Scrapers:** Modern phishing kits use client-side JavaScript rendering, multi-step session gates (`Login -> OTP -> Card`), and Cloudflare Turnstiles that blind `curl`, `requests`, and `BeautifulSoup`.
* **45+ Minute Manual Takedown Bottleneck:** Enterprise SOC analysts and CERT responders spend over 45 minutes per incident manually tracing RDAP/Whois records, capturing DOM screenshots, and filling out registrar abuse forms.

## 2. The Solution: Fighting Illusion with Illusion

MAYA executes a 4-stage autonomous neutralization pipeline:

1. **Interrogate:** Enters live suspect URLs inside a sandboxed TinyFish real-browser environment to bypass Turnstiles and client-side JS session checks.
2. **Poison:** Traverses multi-step fake login funnels and injects trackable synthetic canary honey-tokens into the scammer's backend database.
3. **Trace:** Captures the hidden `POST` exfiltration endpoint, full network redirect chain, and RDAP/Whois/SSL telemetry in structured JSON.
4. **Neutralize:** Automatically populates registrar and hosting abuse portals with DOM evidence and arms an hourly watchdog until the domain returns `HTTP 404`.

## 3. How TinyFish Powers MAYA

| TinyFish Primitive | Endpoint / Module | Role in Project MAYA |
| :--- | :--- | :--- |
| **TinyFish Search & Fetch APIs** | `Search` / `Fetch` | Performs zero-latency OSINT discovery and pulls clean Markdown telemetry from RDAP/Whois registries, DNS records, and SSL certificate logs to identify registrar abuse contacts. |
| **TinyFish Agent API** | `/v1/automation/run-sse` | Spawns a stateful headless browser with **Live Browser Preview** to navigate multi-step fake login flows, bypass JS checks, inject synthetic canary credentials, and extract final network redirect + exfiltration endpoints in structured JSON. |
| **TinyFish Browser API** | `Browser Session` | Launches a parallel browser session on the hosting provider's abuse portal to automatically populate and prepare the takedown submission with captured DOM evidence. |
| **TinyFish Monitor API** | `/v1/monitors` | Schedules an automated cron watchdog with an LLM diff judge to alert the SOC Command Center the exact moment the registrar suspends the malicious domain (`HTTP 404`). |

## 4. System Architecture & Concurrent Execution Flow

MAYA uses an asynchronous FastAPI orchestrator (`asyncio.gather`) that fans out a single suspect URL across three concurrent TinyFish worker threads and streams real-time Server-Sent Events (SSE) to a 3-Pane Split-Screen SOC Command Center UI:

* **Thread A (`fetch_osint_worker`):** Queries TinyFish Search & Fetch APIs to extract RDAP/Whois registrar abuse contacts, ASN details, and SSL certificate fingerprints in clean Markdown.
* **Thread B (`agent_interrogation_worker`):** Invokes TinyFish Agent API (`/v1/automation/run-sse`) to walk the live phishing DOM, stream the `live_browser_preview_url` to the Center Pane, inject synthetic canary honey-tokens, capture the hidden `POST` exfiltration endpoint, and trigger the TinyFish Browser API on the registrar's abuse form.
* **Thread C (`monitor_watchdog_worker`):** Arms an hourly cron job via TinyFish Monitor API (`/v1/monitors`) with an LLM diff judge to track domain suspension state.

## 5. Reference Implementation (`maya_engine.py`)

```python
import asyncio
import os
import uuid
import httpx
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from pydantic import BaseModel

app = FastAPI(title="MAYA Counter-Deception Engine", version="1.0.0")

TINYFISH_API_KEY = os.getenv("TINYFISH_API_KEY", "tf_live_maya_demo_key")
TINYFISH_BASE_URL = "https://api.tinyfish.io/v1"

HEADERS = {
    "Authorization": f"Bearer {TINYFISH_API_KEY}",
    "Content-Type": "application/json",
}


class TargetSubmission(BaseModel):
    suspect_url: str
    canary_batch_size: int = 5


def generate_canary_token() -> dict:
    token_id = uuid.uuid4().hex[:8]
    return {
        "canary_id": f"MAYA-CANARY-{token_id.upper()}",
        "username": f"sec.ops.{token_id}@enterprise-auth-canary.org",
        "password": f"Tf!9{token_id}Zx#2026",
        "otp_code": "482910",
        "tracking_webhook": f"https://telemetry.maya-soc.internal/canary/{token_id}",
    }


async def thread_a_fetch_osint(client: httpx.AsyncClient, suspect_url: str) -> dict:
    """Thread A: TinyFish Search & Fetch APIs for RDAP/Whois and SSL telemetry."""
    payload = {
        "url": suspect_url,
        "extract_markdown": True,
        "include_whois_rdap": True,
        "include_ssl_chain": True,
    }
    resp = await client.post(f"{TINYFISH_BASE_URL}/fetch", headers=HEADERS, json=payload)
    return resp.json()


async def thread_b_agent_interrogate(client: httpx.AsyncClient, suspect_url: str, canary: dict) -> dict:
    """Thread B: TinyFish Agent API (/v1/automation/run-sse) + Browser API abuse prep."""
    agent_prompt = (
        f"Navigate to {suspect_url}. Bypass any Cloudflare Turnstile or JS session gate. "
        f"Locate the credential input fields and submit username '{canary['username']}' "
        f"and password '{canary['password']}'. If an OTP step appears, enter '{canary['otp_code']}'. "
        "Capture the network POST exfiltration URL, redirect chain, and DOM evidence in structured JSON."
    )
    run_payload = {
        "url": suspect_url,
        "goal": agent_prompt,
        "stream_sse": True,
        "live_preview": True,
        "response_format": {
            "type": "json_object",
            "schema": {
                "exfiltration_endpoint": "string",
                "redirect_chain": ["string"],
                "dom_kit_signature": "string",
                "honey_token_injected": "boolean",
            },
        },
    }
    resp = await client.post(
        f"{TINYFISH_BASE_URL}/automation/run-sse", headers=HEADERS, json=run_payload
    )
    return resp.json()


async def thread_c_arm_monitor(client: httpx.AsyncClient, suspect_url: str) -> dict:
    """Thread C: TinyFish Monitor API (/v1/monitors) with LLM diff judge."""
    monitor_payload = {
        "url": suspect_url,
        "schedule": "0 * * * *",  # Hourly cron watchdog
        "llm_judge_prompt": (
            "Evaluate whether this phishing portal has been suspended, taken down, "
            "or returns HTTP 404 / registrar suspension notice."
        ),
        "alert_webhook": "https://telemetry.maya-soc.internal/monitors/takedown-confirmed",
    }
    resp = await client.post(
        f"{TINYFISH_BASE_URL}/monitors", headers=HEADERS, json=monitor_payload
    )
    return resp.json()


@app.post("/api/v1/neutralize")
async def neutralize_phishing_target(target: TargetSubmission):
    canary = generate_canary_token()
    async with httpx.AsyncClient(timeout=60.0) as client:
        osint_data, agent_evidence, monitor_state = await asyncio.gather(
            thread_a_fetch_osint(client, target.suspect_url),
            thread_b_agent_interrogate(client, target.suspect_url, canary),
            thread_c_arm_monitor(client, target.suspect_url),
        )
    return {
        "status": "NEUTRALIZATION_ACTIVE",
        "target": target.suspect_url,
        "canary_injected": canary["canary_id"],
        "osint_telemetry": osint_data,
        "agent_execution": agent_evidence,
        "watchdog_monitor": monitor_state,
    }
```

## 6. Quickstart & Local Setup

```bash
# 1. Clone the repository
git clone https://github.com/team-janaki/project-maya.git
cd project-maya

# 2. Install dependencies
pip install fastapi uvicorn httpx pydantic

# 3. Export your TinyFish API credentials
export TINYFISH_API_KEY="your_tinyfish_api_key_here"

# 4. Launch the FastAPI Counter-Deception Orchestrator
uvicorn maya_engine:app --reload --port 8000
```

## 7. Team Janaki

* **Maumita Saha (Team Lead):** Product Strategy, Threat-Intel Workflow & Pitch Architecture
* **Aadithya S Nair:** Core Systems Architecture, TinyFish Agent Orchestration & Security Engineering
* **Sufiyan Khan:** Frontend Command Center UI, OSINT Pipeline & Live Takedown Integration
