# Golioth Management API — Node.js / TypeScript Patterns

This file covers Node.js and TypeScript usage. All API calls must be made server-side — never from a browser.

## Setup

```bash
npm install node-fetch form-data  # or use native fetch (Node 18+)
npm install dotenv                # optional, for local dev
```

```ts
// env
const API_KEY = process.env.GOLIOTH_API_KEY!;
const PROJECT_ID = process.env.GOLIOTH_PROJECT_ID!;
const ORG_ID = process.env.GOLIOTH_ORG_ID!;
const BASE = 'https://api.golioth.io/v1';
```

## Typed Response Shapes

These cover the most commonly used fields. Add more as needed.

```ts
interface GoliothResponse<T> {
  result: T;
}

interface PagedResponse<T> {
  list: T[];
  page: { perPage: number; page: number; total: number };
}

interface Device {
  id: string;
  name: string;
  status: 'online' | 'offline' | 'never_connected';
  hardwareIds: string[];
  tagIds: string[];
  blueprintId?: string;
  createdAt: string;
  updatedAt: string;
}

interface StreamEntry {
  timestamp: string;
  deviceId: string;
  data: Record<string, unknown>;
}

interface Artifact {
  id: string;
  version: string;
  package: string;
  blueprintId: string;
  createdAt: string;
}

interface Cohort {
  id: string;
  name: string;
  deviceIds: string[];
  tagIds: string[];
}

interface Deployment {
  id: string;
  name: string;
  packages: Array<{ packageId: string; artifactId: string; version: string }>;
}

interface OtaEvent {
  id: string;
  deviceId: string;
  eventType: string;
  status: 'pending' | 'downloading' | 'applying' | 'completed' | 'failed';
  createdAt: string;
}
```

## Helper

A thin wrapper around fetch to handle auth, JSON parsing, and errors in one place:

```ts
async function golioth<T>(
  path: string,
  options: RequestInit = {}
): Promise<T> {
  const res = await fetch(`${BASE}${path}`, {
    ...options,
    headers: {
      'x-api-key': API_KEY,
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });
  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new Error(`Golioth API error ${res.status}: ${JSON.stringify(err)}`);
  }
  const body = await res.json() as GoliothResponse<T>;
  return body.result;
}
```

Usage:

```ts
const devices = await golioth<PagedResponse<Device>>(
  `/projects/${PROJECT_ID}/devices?perPage=100`
);
console.log(devices.list);
```

## Common Workflows

### List devices

```ts
const { list: devices } = await golioth<PagedResponse<Device>>(
  `/projects/${PROJECT_ID}/devices?perPage=100`
);
```

### Query stream (time-series telemetry)

```ts
const end = new Date().toISOString();
const start = new Date(Date.now() - 60 * 60 * 1000).toISOString(); // 1 hour ago

const { list: entries } = await golioth<PagedResponse<StreamEntry>>(
  `/projects/${PROJECT_ID}/devices/${deviceId}/stream?start=${start}&end=${end}&perPage=100`
);
```

### Read / write LightDB State

```ts
// Read a path
const value = await golioth<unknown>(
  `/projects/${PROJECT_ID}/devices/${deviceId}/lightdb/led/status`
);

// Write to a path
await golioth<unknown>(
  `/projects/${PROJECT_ID}/devices/${deviceId}/lightdb/led`,
  {
    method: 'PUT',
    body: JSON.stringify({ brightness: 75, mode: 'pwm' }),
  }
);
```

### Read / write settings

```ts
// List device settings (shows effective value + source)
const settings = await golioth<unknown>(
  `/projects/${PROJECT_ID}/devices/${deviceId}/settings`
);

// Set a per-device setting override
await golioth<unknown>(
  `/projects/${PROJECT_ID}/devices/${deviceId}/settings/LED_BRIGHTNESS`,
  { method: 'PUT', body: JSON.stringify(75) }
);

// Set a project-level default (applies to all devices)
await golioth<unknown>(
  `/organizations/${ORG_ID}/projects/${PROJECT_ID}/config/LED_BRIGHTNESS`,
  { method: 'PUT', body: JSON.stringify(50) }
);
```

### Call an RPC

```ts
const result = await golioth<unknown>(
  `/projects/${PROJECT_ID}/devices/${deviceId}/rpc?timeout=10`,
  {
    method: 'POST',
    body: JSON.stringify({ method: 'reboot', params: [] }),
  }
);
```

### OTA firmware deployment (full flow)

Artifact upload requires `multipart/form-data`. Use the `form-data` package or `FormData` (Node 18+):

```ts
import FormData from 'form-data';
import fs from 'fs';

// Step 1: Upload artifact
const form = new FormData();
form.append('projectId', PROJECT_ID);
form.append('version', '1.2.0');
form.append('blueprintId', blueprintId);
form.append('package', 'main');
form.append('artifact', fs.createReadStream('build/zephyr/zephyr.signed.bin'));

const uploadRes = await fetch(`${BASE}/artifacts`, {
  method: 'POST',
  headers: { 'x-api-key': API_KEY, ...form.getHeaders() },
  body: form,
});
const { result: artifact } = await uploadRes.json() as GoliothResponse<Artifact>;
const artifactId = artifact.id;

// Step 2: Ensure package exists
const { list: packages } = await golioth<PagedResponse<{ id: string; name: string }>>(
  `/organizations/${ORG_ID}/projects/${PROJECT_ID}/packages`
);
let packageId = packages.find(p => p.name === 'main')?.id;
if (!packageId) {
  const pkg = await golioth<{ id: string }>(
    `/organizations/${ORG_ID}/projects/${PROJECT_ID}/packages`,
    { method: 'POST', body: JSON.stringify({ name: 'main' }) }
  );
  packageId = pkg.id;
}

// Step 3: Get or create cohort
const cohort = await golioth<Cohort>(
  `/organizations/${ORG_ID}/projects/${PROJECT_ID}/cohorts`,
  {
    method: 'POST',
    body: JSON.stringify({ name: 'production-fleet', deviceIds: [deviceId] }),
  }
);

// Step 4: Create deployment
const deployment = await golioth<Deployment>(
  `/organizations/${ORG_ID}/projects/${PROJECT_ID}/cohorts/${cohort.id}/deployments`,
  {
    method: 'POST',
    body: JSON.stringify({
      name: 'v1.2.0-rollout',
      packages: [{ packageId, artifactId, version: '1.2.0' }],
    }),
  }
);

// Step 5: Poll for completion
async function waitForOta(deviceId: string, timeoutMs = 120_000) {
  const deadline = Date.now() + timeoutMs;
  while (Date.now() < deadline) {
    const { list: events } = await golioth<PagedResponse<OtaEvent>>(
      `/organizations/${ORG_ID}/projects/${PROJECT_ID}/ota-events?deviceId=${deviceId}&limit=5`
    );
    const latest = events[0];
    if (latest?.status === 'completed') return latest;
    if (latest?.status === 'failed') throw new Error(`OTA failed: ${JSON.stringify(latest)}`);
    await new Promise(r => setTimeout(r, 5000)); // poll every 5s
  }
  throw new Error('OTA timed out');
}
```

## Express Route Example

A typical backend proxy pattern — keep the API key server-side, expose a safe endpoint to your frontend:

```ts
import express from 'express';

const app = express();

app.get('/api/devices', async (req, res) => {
  try {
    const data = await golioth<PagedResponse<Device>>(
      `/projects/${PROJECT_ID}/devices?perPage=100`
    );
    res.json(data.list);
  } catch (err) {
    res.status(500).json({ error: 'Failed to fetch devices' });
  }
});

app.get('/api/devices/:id/stream', async (req, res) => {
  const end = new Date().toISOString();
  const start = new Date(Date.now() - 60 * 60 * 1000).toISOString();
  try {
    const data = await golioth<PagedResponse<StreamEntry>>(
      `/projects/${PROJECT_ID}/devices/${req.params.id}/stream?start=${start}&end=${end}&perPage=100`
    );
    res.json(data.list);
  } catch (err) {
    res.status(500).json({ error: 'Failed to fetch stream' });
  }
});
```

## Next.js API Route Example

Next.js API routes run server-side, so they're the right place to call Golioth. Do not call the Golioth API from React components or client-side code.

```ts
// app/api/devices/route.ts  (Next.js 13+ App Router)
import { NextResponse } from 'next/server';

export async function GET() {
  const devices = await golioth<PagedResponse<Device>>(
    `/projects/${process.env.GOLIOTH_PROJECT_ID}/devices?perPage=100`
  );
  return NextResponse.json(devices.list);
}
```

## Notes

- **Multipart upload** (artifact upload) can't use the `golioth()` helper directly because it needs custom `Content-Type` headers from `form-data`. Always use `fetch` directly for that step.
- **Pagination**: default `perPage` is 10. Always pass `?perPage=100` unless you need pagination logic.
- **Offline devices**: settings and LightDB writes succeed even if the device is offline — it picks them up on reconnect. RPC requires the device to be online.
- **Environment variables**: In production, use your platform's secrets management (Vercel env vars, AWS Secrets Manager, etc.) rather than `.env` files.
