# Golioth Management API — Common Workflows

Assumes environment variables are set:
```bash
export GOLIOTH_API_KEY="your-api-key"
export GOLIOTH_PROJECT_ID="your-project-id"
export GOLIOTH_ORG_ID="your-organization-id"
BASE="https://api.golioth.io/v1"
```

---

## 1. OTA Firmware Update (Full Flow)

This is the most common multi-step workflow. It goes: upload artifact → ensure a package exists → ensure a cohort exists → create a deployment → monitor events.

### Step 1: Upload the firmware binary

```bash
ARTIFACT=$(curl -s -X POST \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -F "projectId=$GOLIOTH_PROJECT_ID" \
  -F "version=1.2.0" \
  -F "blueprintId=$BLUEPRINT_ID" \
  -F "package=main" \
  -F "artifact=@build/zephyr/zephyr.signed.bin" \
  "$BASE/artifacts")

ARTIFACT_ID=$(echo $ARTIFACT | jq -r '.result.id')
echo "Artifact ID: $ARTIFACT_ID"
```

### Step 2: Get or create a package

```bash
# List existing packages
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/packages" \
  | jq '.result.list[] | {id, name}'

# Create a package if needed
PACKAGE=$(curl -s -X POST \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "main"}' \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/packages")

PACKAGE_ID=$(echo $PACKAGE | jq -r '.result.id')
```

### Step 3: Get or create a cohort

```bash
# List existing cohorts
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/cohorts" \
  | jq '.result.list[] | {id, name}'

# Create a cohort targeting specific devices
COHORT=$(curl -s -X POST \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"led-pwm-test\", \"deviceIds\": [\"$DEVICE_ID\"]}" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/cohorts")

COHORT_ID=$(echo $COHORT | jq -r '.result.id')
```

### Step 4: Create a deployment

```bash
curl -s -X POST \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"v1.2.0-rollout\",
    \"packages\": [{
      \"packageId\": \"$PACKAGE_ID\",
      \"artifactId\": \"$ARTIFACT_ID\",
      \"version\": \"1.2.0\"
    }]
  }" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/cohorts/$COHORT_ID/deployments"
```

### Step 5: Monitor OTA events

```bash
# Poll for events — filter by deviceId and cohortId
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/ota-events?deviceId=$DEVICE_ID&limit=20" \
  | jq '.result.list[] | {eventType, status, createdAt}'
```

Status values: `pending` → `downloading` → `applying` → `completed` (or `failed`)

---

## 2. Reading and Writing Settings

Settings exist at two levels: **project** (default for all devices) and **device** (override for one device).
The device-level value always wins. There is no blueprint-level settings API.

The device SDK automatically receives setting updates and applies them via registered callbacks.

### Project-level settings (defaults for all devices)

```bash
# List all project settings
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/config" \
  | jq '.result'

# Set a project-level default
curl -s -X PUT \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '50' \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/config/LED_BRIGHTNESS"
```

### Device-level settings (per-device overrides)

```bash
# List all settings for a device (shows effective value + whether it came from project or device)
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices/$DEVICE_ID/settings" \
  | jq '.result'

# Set a per-device override (string value)
curl -s -X PUT \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '"pwm_dim"' \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices/$DEVICE_ID/settings/LED_MODE"

# Set a numeric override (e.g. PWM duty cycle 0-100)
curl -s -X PUT \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '75' \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices/$DEVICE_ID/settings/LED_BRIGHTNESS"

# Remove override — device falls back to project default
curl -s -X DELETE \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices/$DEVICE_ID/settings/LED_MODE"
```

### Targeting multiple devices with a setting

There's no single API call to apply a setting to a group. Instead, filter devices by whatever
criteria makes sense for your use case (tag, blueprint, etc.) and apply the setting to each:

```bash
# Apply LED_BRIGHTNESS=75 to all devices with tag "led-capable"
DEVICES=$(curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices?perPage=100" \
  | jq -r '.result.list[] | select(.tags[] | .name == "led-capable") | .id')

for DEVICE_ID in $DEVICES; do
  curl -s -X PUT \
    -H "x-api-key: $GOLIOTH_API_KEY" \
    -H "Content-Type: application/json" \
    -d '75' \
    "$BASE/projects/$GOLIOTH_PROJECT_ID/devices/$DEVICE_ID/settings/LED_BRIGHTNESS"
done
```

Note: cohorts exist for OTA targeting, not settings. For settings, the application controls targeting logic.

---

## 3. Reading Device Stream Data (Telemetry)

```bash
# Get the last 10 stream entries
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices/$DEVICE_ID/stream?perPage=10" \
  | jq '.result.list[] | {timestamp, data}'

# Filter by time range (RFC3339 timestamps)
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices/$DEVICE_ID/stream?start=2025-01-01T00:00:00Z&end=2025-01-02T00:00:00Z" \
  | jq '.result.list'
```

---

## 4. Reading and Writing LightDB State

LightDB State is a JSON document that stays synced between device and cloud.

```bash
# Read the full state document
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices/$DEVICE_ID/lightdb" \
  | jq '.result'

# Read a specific path
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices/$DEVICE_ID/lightdb/led/status" \
  | jq '.result'

# Write to a path (device will be notified)
curl -s -X PUT \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"brightness": 75, "mode": "pwm"}' \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices/$DEVICE_ID/lightdb/led"
```

---

## 5. Calling an RPC on a Device

RPC lets you invoke a function on the device and get a response.

```bash
# Call 'reboot' with no params
curl -s -X POST \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"method": "reboot", "params": []}' \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices/$DEVICE_ID/rpc?timeout=10" \
  | jq '.'

# Call a custom function with arguments
curl -s -X POST \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"method": "set_led", "params": [75, "pwm"]}' \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices/$DEVICE_ID/rpc?timeout=15"
```

The device must have an RPC handler registered for the method name. If the device is offline or doesn't respond within `timeout` seconds, the request returns an error.

---

## 6. Creating and Managing Data Pipelines

Pipelines route device stream data to external destinations. They are powerful and underused — you can dynamically redirect data, chain transformations, and toggle routing on/off without deleting the pipeline.

### Critical: the pipeline field is base64-encoded YAML

The REST API does not accept a JSON pipeline definition. The pipeline configuration is written in **YAML**, then **base64-encoded**, and sent as the `pipeline` string field in the JSON body. Sending raw YAML or a JSON object in that field will fail.

```
POST body: { "name": "...", "enabled": true, "pipeline": "<base64(yaml)>" }
```

### Pipeline YAML structure

```yaml
steps:
  - name: step-0
    transformer:           # optional — transforms the payload before passing on
      type: cbor-to-json   # e.g. if device sends CBOR
      version: v1
  - name: step-1
    destination:           # optional — delivers data to an external service
      type: webhook
      version: v1
      parameters:
        url: https://your-endpoint.example.com/ingest
        headers:
          x-api-key: $MY_WEBHOOK_SECRET
```

Each step can have a transformer, a destination, or both. When a step has both, the transformed data goes to that step's destination and the *original* (pre-transform) data continues to the next step.

### Create a pipeline (curl)

```bash
# Write your pipeline YAML to a file, then base64-encode it
cat > /tmp/pipeline.yaml << 'EOF'
steps:
  - name: forward-to-webhook
    destination:
      type: webhook
      version: v1
      parameters:
        url: https://your-endpoint.example.com/ingest
EOF

ENCODED=$(base64 -w 0 /tmp/pipeline.yaml)

curl -s -X POST \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"sensor-webhook\", \"enabled\": true, \"pipeline\": \"$ENCODED\"}" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/pipelines" \
  | jq '.result.id'
```

### Enable / disable a pipeline (the data spout)

You can toggle a pipeline on and off without deleting it — useful for pausing data routing to a destination:

```bash
# Disable (stop data flow)
curl -s -X PUT \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"enabled": false}' \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/pipelines/$PIPELINE_ID"

# Re-enable
curl -s -X PUT \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"enabled": true}' \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/pipelines/$PIPELINE_ID"
```

### Update a pipeline's YAML

Same as create but use PUT with the pipeline ID:

```bash
ENCODED=$(base64 -w 0 /tmp/updated-pipeline.yaml)

curl -s -X PUT \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"sensor-webhook\", \"enabled\": true, \"pipeline\": \"$ENCODED\"}" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/pipelines/$PIPELINE_ID"
```

### List pipelines and check errors

```bash
# List all pipelines
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/pipelines" \
  | jq '.result.list[] | {id, name, enabled}'

# Check for pipeline delivery errors
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/pipelines/errors" \
  | jq '.result'
```

### Available destinations

`webhook`, `influxdb`, `mongodb`, `aws-s3`, `aws-sqs`, `aws-kinesis`, `gcp-pubsub`, `gcp-cloud-storage`, `azure-event-hubs`, `azure-blob-storage`, `kafka`, `lightdb-stream`, `lightdb-state`, `memfault`, `logs`, `batch`

### Available transformers

`cbor-to-json`, `json-patch`, `inject-metadata`, `inject-path`, `extract-timestamp`, `struct-to-json`, `base64`, `embed-in-json`

### Multi-step example: decode CBOR then forward

```yaml
steps:
  - name: decode
    transformer:
      type: cbor-to-json
      version: v1
  - name: forward
    destination:
      type: influxdb
      version: v1
      parameters:
        url: https://us-east-1-1.aws.cloud2.influxdata.com
        token: $INFLUXDB_TOKEN
        bucket: device-data
        measurement: sensors
```

### Switching a pipeline from JSON to CBOR (device format change)

When a device switches from sending JSON to sending CBOR (e.g. for bandwidth efficiency), any existing pipeline will stop delivering data correctly — the destination receives raw CBOR bytes it can't parse. Fix this by adding a `cbor-to-json` transformer step *before* the destination step:

```yaml
steps:
  - name: decode-cbor
    transformer:
      type: cbor-to-json
      version: v1
  - name: forward-to-webhook
    destination:
      type: webhook
      version: v1
      parameters:
        url: https://your-endpoint.example.com/ingest
```

Update the existing pipeline (base64-encode the new YAML and PUT it):

```bash
cat > /tmp/pipeline-cbor.yaml << 'EOF'
steps:
  - name: decode-cbor
    transformer:
      type: cbor-to-json
      version: v1
  - name: forward-to-webhook
    destination:
      type: webhook
      version: v1
      parameters:
        url: https://your-endpoint.example.com/ingest
EOF

ENCODED=$(base64 -w 0 /tmp/pipeline-cbor.yaml)

curl -s -X PUT \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"sensor-webhook\", \"enabled\": true, \"pipeline\": \"$ENCODED\"}" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/pipelines/$PIPELINE_ID"
```

### Troubleshooting: pipeline errors endpoint

When data stops arriving at a destination, check the errors endpoint first:

```bash
# All errors across all pipelines
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/pipelines/errors" \
  | jq '.result'

# Errors for a specific pipeline
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/organizations/$GOLIOTH_ORG_ID/projects/$GOLIOTH_PROJECT_ID/pipelines/$PIPELINE_ID/errors" \
  | jq '.result'
```

Common failure causes:
- **CBOR data reaching a destination that expects JSON** — add `cbor-to-json` transformer before the destination
- **Webhook returning non-2xx** — check destination URL and auth headers in the pipeline YAML
- **Disabled pipeline** — check `enabled` flag; data flows through Golioth fine but won't be forwarded
- **Wrong pipeline YAML encoding** — if the pipeline was created/updated with raw YAML instead of base64, re-upload with proper base64 encoding

---

## 7. Listing and Finding Devices

```bash
# List all devices
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices?perPage=100" \
  | jq '.result.list[] | {id, name, status}'

# Find a device by name (client-side filter)
curl -s -H "x-api-key: $GOLIOTH_API_KEY" \
  "$BASE/projects/$GOLIOTH_PROJECT_ID/devices" \
  | jq '.result.list[] | select(.name == "my-device-001")'
```

---

## End-to-End Example: Build, Deploy, Test a New Firmware Feature

This is the full workflow when combining the Golioth device SDK skill with this skill.

> **Note:** Steps involving Zephyr/west (building firmware, flashing, writing device-side handlers)
> are handled by the **Golioth device SDK skill**. If that skill is not installed, recommend the
> user install it before proceeding with the firmware side of the workflow.

1. **Write firmware** (device SDK skill): implement the new feature and setting handler in Zephyr
2. **Build the image** (device SDK skill): compile for the target board
3. **Upload artifact** (this skill): POST the built binary to `/v1/artifacts`
4. **Create/update deployment** (this skill): link cohort → package → artifact
5. **Wait for OTA** (this skill): poll `/ota-events?deviceId=...` until status is `completed`
6. **Test via setting** (this skill): PUT to `/settings/LED_BRIGHTNESS` with the new value
7. **Verify via stream** (this skill): GET `/stream` to confirm the device reported the expected result

---

## Tips

- Use `jq` to parse responses: `curl ... | jq '.result'`
- Store IDs in shell variables across steps to avoid copy-paste errors
- The `perPage` default is usually 10; use `?perPage=100` to get more results
- If a device is offline, settings/LightDB writes still succeed — the device picks them up on reconnect
- RPC requires the device to be online; use the `timeout` param accordingly
