# Golioth OTA — Building Custom Rollout and Monitoring UIs

This file covers patterns for building custom UIs on top of the Golioth OTA API: staged rollouts, fleet version monitoring, and tracking deployment progress.

## Entity model

```
Blueprint ──── defines hardware type
    │
Artifact ────── firmware binary (version + package + blueprintId)
    │
Package ─────── named firmware component (e.g. "main", "radio-firmware")
    │
Deployment ──── links artifact(s) to a cohort { name, artifactIds[] }
    │
Cohort ──────── named device group; has one active deployment at a time
    │
Device ──────── belongs to one cohort; receives cohort's active deployment
```

Key constraints:
- A device belongs to **one cohort at a time**
- A cohort has **one active deployment at a time** (creating a new deployment replaces the old one for that cohort)
- Multiple cohorts can have deployments from the same artifact at different times — this is how staged rollouts work

## Staged rollout pattern

The standard pattern is three cohorts of increasing size:

```
test-cohort       (5–10 devices, your dev/QA hardware)
    ↓  monitor + verify
beta-cohort       (10–20% of fleet)
    ↓  monitor + verify
production-cohort (remaining devices)
```

### Step 1: Upload the artifact once

```bash
ARTIFACT=$(curl -s -X POST \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -F "projectId=$GOLIOTH_PROJECT_ID" \
  -F "version=2.0.0" \
  -F "blueprintId=$BLUEPRINT_ID" \
  -F "package=main" \
  -F "artifact=@build/zephyr/zephyr.signed.bin" \
  "https://api.golioth.io/v1/artifacts")

ARTIFACT_ID=$(echo $ARTIFACT | jq -r '.result.id')
```

### Step 2: Deploy to test cohort

```bash
# Get your test cohort ID
TEST_COHORT_ID=$(curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "https://api.golioth.io/v1/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/cohorts?perPage=100" \
  | jq -r '.result.list[] | select(.name == "test-cohort") | .id')

# Deploy to test cohort
curl -s -X POST \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"v2.0.0-test\",
    \"packages\": [{
      \"packageId\": \"$PACKAGE_ID\",
      \"artifactId\": \"$ARTIFACT_ID\",
      \"version\": \"2.0.0\"
    }]
  }" \
  "https://api.golioth.io/v1/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/cohorts/$TEST_COHORT_ID/deployments"
```

### Step 3: Monitor the test cohort

Poll OTA events for the cohort. Each event has `currentVersion`, `targetVersion`, and `status`:

```bash
# Poll events for the test cohort
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "https://api.golioth.io/v1/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/ota-events?cohortId=$TEST_COHORT_ID&limit=50" \
  | jq '.result.list[] | {deviceId, packageId, currentVersion, targetVersion, status, eventType, timestamp}'
```

Wait until all devices in the cohort reach `status: completed` before promoting.

To check if a cohort is fully updated, compare `deviceCount` (from the cohort object) against how many distinct `deviceId`s have `status: completed` for `targetVersion == "2.0.0"`:

```bash
# Count completed devices for this version
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "https://api.golioth.io/v1/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/ota-events?cohortId=$TEST_COHORT_ID&limit=100" \
  | jq '[.result.list[] | select(.targetVersion == "2.0.0" and .status == "completed")] | length'

# Get total device count for cohort
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "https://api.golioth.io/v1/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/cohorts/$TEST_COHORT_ID" \
  | jq '.result.deviceCount'
```

### Step 4: Promote to beta, then production

Once satisfied, repeat the deployment step for each subsequent cohort using the same `ARTIFACT_ID`:

```bash
# Promote to beta
curl -s -X POST \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"v2.0.0-beta\",
    \"packages\": [{
      \"packageId\": \"$PACKAGE_ID\",
      \"artifactId\": \"$ARTIFACT_ID\",
      \"version\": \"2.0.0\"
    }]
  }" \
  "https://api.golioth.io/v1/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/cohorts/$BETA_COHORT_ID/deployments"

# Monitor beta, then promote to production the same way
```

## OTA event fields for monitoring

Each OTA event from `GET /ota-events`:

```json
{
  "deviceId": "abc123",
  "packageId": "pkg456",
  "eventType": "...",
  "currentVersion": "1.5.0",
  "targetVersion": "2.0.0",
  "status": 0,
  "message": "...",
  "timestamp": "2026-03-27T10:00:00Z"
}
```

- `currentVersion` — what the device is running right now
- `targetVersion` — what the deployment is trying to install
- `status` — integer; use `eventType` as the human-readable progression signal
- `eventType` — progression: `"queued"` → `"downloading"` → `"applying"` → `"completed"` (or `"failed"`)

Use `startTime` / `endTime` query params (RFC3339) to scope events to a recent window when polling.

## OTA stats endpoint

For a rollout progress summary across a cohort (good for a progress bar):

```bash
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "https://api.golioth.io/v1/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/ota-events/stats?cohortId=$TEST_COHORT_ID" \
  | jq '.result'
```

## Per-device firmware version

Each device object includes a `metadata` field with an `update` sub-object containing the latest firmware update information. Use this to show the current firmware version per device in a fleet table:

```bash
# Get current firmware state for all devices
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "https://api.golioth.io/v1/projects/$GOLIOTH_PROJECT_ID/devices?perPage=100" \
  | jq '.result.list[] | {id, name, status, firmware: .metadata.update}'
```

## Building a rollout UI: recommended data flow

```
1. List cohorts → display cohort names, deviceCount, activeDeploymentId
2. For each cohort with an active deployment:
   a. GET /ota-events?cohortId=X&limit=100 → latest events per device
   b. Compare targetVersion to deployment version for progress %
   c. Count by eventType to show queued/downloading/applying/completed/failed breakdown
3. Poll every 10–30s to keep UI live (WebSockets are deprecated — polling is the approach)
4. Show per-device row: deviceId, currentVersion, targetVersion, last eventType
```

## Listing cohorts with deployment status

```bash
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "https://api.golioth.io/v1/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/cohorts?perPage=100" \
  | jq '.result.list[] | {id, name, deviceCount, activeDeploymentId}'
```

## Rollback

To roll back a cohort, create a new deployment pointing to the previous artifact version:

```bash
curl -s -X POST \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"v1.5.0-rollback\",
    \"packages\": [{
      \"packageId\": \"$PACKAGE_ID\",
      \"artifactId\": \"$PREVIOUS_ARTIFACT_ID\",
      \"version\": \"1.5.0\"
    }]
  }" \
  "https://api.golioth.io/v1/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/cohorts/$COHORT_ID/deployments"
```
