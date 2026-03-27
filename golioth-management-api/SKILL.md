---
name: golioth-management-api
description: >
  Use the Golioth Management API (https://api.golioth.io/v1) to manage IoT
  device fleets — devices, OTA firmware deployments, settings, LightDB state,
  data streams, pipelines, and remote procedure calls. Also use when combining
  fleet management with the Golioth device SDK skill for end-to-end workflows.
---

# Golioth Management API Skill

## Authentication

All requests require a project-scoped API key passed via the `x-api-key` header.

```bash
curl -H "x-api-key: $GOLIOTH_API_KEY" https://api.golioth.io/v1/projects
```

**Getting an API key**: Create one in the [Golioth console](https://console.golioth.io) under Project Settings > API Keys, or via `goliothctl project apikeys create`.

Store it as an environment variable (`GOLIOTH_API_KEY`) — never hardcode it.

**Two ID types you'll need frequently:**
- `projectId` — your project's ID (e.g. `my-project`)
- `organizationId` — your org's ID; required for cohort/deployment/pipeline/OTA endpoints

Both are visible in the console URL and via `GET /v1/projects` / `GET /v1/organizations`.

## URL Patterns

Two base path patterns exist — which one to use depends on the resource:

| Pattern | Used for |
|---------|----------|
| `/v1/projects/{projectId}/...` | Devices, settings, LightDB, stream, RPC, artifacts, blueprints, certificates, API keys |
| `/v1/organizations/{organizationId}/projects/{projectId}/...` | Cohorts, deployments, packages, pipelines, PKI, OTA events |

When in doubt, check `references/endpoints.md` for the exact path.

## Key Concepts

- **Device**: A physical IoT device registered in Golioth. Has credentials, tags, and a blueprint.
- **Blueprint**: A device template (hardware type/platform). Devices belong to a blueprint.
- **Artifact**: A firmware binary uploaded to Golioth. Has a version, package name, and blueprint association.
- **Package**: A named firmware component (e.g. `main`, `radio-firmware`). Groups artifacts by purpose.
- **Cohort**: A group of devices targeted for a specific firmware deployment.
- **Deployment**: Links a cohort to a set of package/artifact versions to deliver via OTA.
- **Pipeline**: A data routing rule that processes device stream data (transform, forward). Pipelines are defined in YAML and can be toggled on/off via the `enabled` flag without deleting them. **Important**: the pipeline YAML must be base64-encoded when sent via the REST API — it goes in the `pipeline` field as a base64 string, not raw YAML or JSON. See `references/workflows.md` section 6 for the full pattern.
- **Settings**: Key-value config pushed to devices. Project-level settings are defaults for all devices; device-level settings override them for a specific device. There is no blueprint-level settings API.
- **LightDB State**: A JSON document synced between the cloud and the device (like a digital twin).
- **Stream**: Time-series telemetry data sent from devices to the cloud.

## Common Workflows

See `references/workflows.md` for step-by-step guides on:
- OTA firmware update (full flow)
- Reading and writing device settings
- Querying device stream data
- Setting up a data pipeline
- Calling an RPC on a device

## Language Context

Adapt examples to match the user's environment:

- **Node.js / TypeScript / JavaScript** (Express, Next.js, Bun, Deno, etc.) → read `references/node.md` for typed helpers, workflow patterns, and a Next.js API route example. Use this whenever the user mentions any JS/TS runtime or framework, or when they're building a web backend.
- **Python** (FastAPI, Flask, Django, scripts) → translate curl examples to `httpx` or `requests` with `headers={"x-api-key": os.environ["GOLIOTH_API_KEY"]}`. The same URL patterns and response shapes apply.
- **Shell / exploration / no clear context** → use `curl` examples as shown below.

When language context is ambiguous, default to `curl` but mention that Node.js and Python patterns are available.

## Making API Calls

Use `curl` for quick one-off calls, or write a script in Python/TypeScript/Bash for multi-step workflows.

**Response format**: All successful responses wrap data in a `result` key:
```json
{ "result": { ... } }
```

**Pagination**: List endpoints return a `page` object. Use `?page=0&perPage=100` query params.

**Error handling**: Non-2xx responses include a `code` and `message`. Always check the status.

Example — list all devices in a project:
```bash
curl -s \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  "https://api.golioth.io/v1/projects/$GOLIOTH_PROJECT_ID/devices" \
  | jq '.result.list'
```

## Building a Custom Backend/Frontend

### Architecture: never call this API directly from a browser

The Golioth Management API is a **server-side API**. A CORS policy is in place that will block direct browser requests, so a frontend-only app (plain JS, React, etc.) cannot call it directly — and even if it could, doing so would be a serious security mistake.

The correct architecture is:

```
Browser/Frontend  →  Your Backend Server  →  Golioth Management API
```

Your backend holds the API key and proxies requests to Golioth on behalf of your frontend. The frontend never sees the key.

### API key security

The `x-api-key` grants **full access to every function of your fleet** — devices, settings, OTA deployments, pipelines, everything. Treat it like a root password:

- Store it only in server-side environment variables or a secrets manager
- Never embed it in frontend code, mobile apps, or public repositories
- Never log it or include it in error messages returned to clients
- Rotate it immediately if it is ever exposed

### Live data: polling

WebSocket support has been deprecated. The main method for live data in a custom dashboard is **polling** the relevant endpoints (`/stream`, `/ota-events`, `/lightdb`, etc.) on an interval.

Keep polling intervals reasonable — the maximum page size is 100 results per request. Design your backend to cache or aggregate data where possible rather than hammering the API on every frontend request.

## Device Format and Pipeline Alignment

When a device switches its stream encoding (e.g. JSON → CBOR for bandwidth efficiency), the pipeline must be updated to match — otherwise the destination receives unreadable bytes. The fix is to add a `cbor-to-json` transformer step before the destination in the pipeline YAML.

This is a common cross-skill scenario: the **Golioth device SDK skill** handles the firmware-side change (encoding format, stream path), while this skill handles updating the pipeline to match. If a user says their webhook/database stopped receiving data after a firmware update, suspect a format mismatch and check the pipeline errors endpoint first.

## Related Skills

This skill covers cloud-side fleet management only. Anything involving Zephyr, west, device firmware,
or device-side SDK code belongs to the **Golioth device SDK skill**. If the user asks about building
firmware, writing setting handlers, flashing a device, or any other device-side topic and the device
SDK skill is not installed, recommend they install it before proceeding.

## Reference Files

- `references/endpoints.md` — Complete API endpoint reference organized by resource
- `references/workflows.md` — Step-by-step workflow guides for common tasks (curl/bash)
- `references/node.md` — Node.js/TypeScript patterns: typed helpers, all workflows, Express and Next.js examples
- `references/ota-ui.md` — Building custom OTA UIs: staged rollout patterns, fleet version monitoring, OTA event fields, rollback. Read this when the user is building a dashboard, rollout UI, or wants to monitor a deployment in progress.
