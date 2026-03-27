# Golioth Management API — Endpoint Reference

Base URL: `https://api.golioth.io`
Auth header: `x-api-key: $GOLIOTH_API_KEY`

---

## Organizations & Projects

```
GET    /v1/organizations                          List organizations
POST   /v1/organizations                          Create organization
GET    /v1/organizations/{organizationId}         Get organization
PUT    /v1/organizations/{organizationId}         Update organization

GET    /v1/projects                               List projects
POST   /v1/projects                               Create project
GET    /v1/projects/{projectId}                   Get project
PUT    /v1/projects/{projectId}                   Update project
DELETE /v1/projects/{projectId}                   Delete project
```

---

## Devices

```
GET    /v1/projects/{projectId}/devices                       List devices
POST   /v1/projects/{projectId}/devices                       Create device
GET    /v1/projects/{projectId}/devices/{deviceId}            Get device
PUT    /v1/projects/{projectId}/devices/{deviceId}            Replace device
PATCH  /v1/projects/{projectId}/devices/{deviceId}            Update device
DELETE /v1/projects/{projectId}/devices/{deviceId}            Delete device
```

Query params for list: `page`, `perPage`, `tagFilter`, `blueprintId`, `status`

Create device body:
```json
{
  "name": "my-device",
  "hardwareIds": ["AA:BB:CC:DD:EE:FF"],
  "tagIds": [],
  "blueprintId": "optional-blueprint-id"
}
```

---

## Device Credentials

```
POST   /v1/projects/{projectId}/credentials                   Add credential (PSK or cert)
DELETE /v1/projects/{projectId}/credentials/{credentialId}    Delete credential
```

---

## Settings

Settings exist at two levels. A device-level setting always overrides the project default.

### Project-level settings

```
GET    /v1/organizations/{organizationId}/projects/{projectId}/config                        List all project settings
GET    /v1/organizations/{organizationId}/projects/{projectId}/config/{settingId}            Get a project setting
PUT    /v1/organizations/{organizationId}/projects/{projectId}/config/{settingId}            Set a project setting
```

The `settingId` here is the setting key name. Value body format is the same as device-level (raw JSON).

### Device-level settings (per-device overrides)

A device-level setting overrides the project default for that device only. To target a group of devices,
iterate over them using device list filters (by tag, blueprint, etc.) and apply the setting to each.

```
GET    /v1/projects/{projectId}/devices/{deviceId}/settings                      List device settings (shows effective value + source)
GET    /v1/projects/{projectId}/devices/{deviceId}/settings/{settingName}        Get a specific setting
PUT    /v1/projects/{projectId}/devices/{deviceId}/settings/{settingName}        Set a per-device override
DELETE /v1/projects/{projectId}/devices/{deviceId}/settings/{settingName}        Remove override (falls back to project default)
```

Setting value is sent as a raw JSON value in the body (string, number, bool, object):
```bash
# Set a string setting
curl -X PUT \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '"bright"' \
  "https://api.golioth.io/v1/projects/$PROJECT_ID/devices/$DEVICE_ID/settings/LED_MODE"

# Set a numeric setting
curl -X PUT \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '75' \
  "https://api.golioth.io/v1/projects/$PROJECT_ID/devices/$DEVICE_ID/settings/LED_BRIGHTNESS"
```

---

## LightDB State

```
GET    /v1/projects/{projectId}/devices/{deviceId}/lightdb          Get full state document
GET    /v1/projects/{projectId}/devices/{deviceId}/lightdb/{path}   Get value at path
PUT    /v1/projects/{projectId}/devices/{deviceId}/lightdb/{path}   Set value at path
DELETE /v1/projects/{projectId}/devices/{deviceId}/lightdb/{path}   Delete path
```

Path uses `/` separators (e.g. `/sensor/temperature`). Value is raw JSON in body.

---

## Stream (Telemetry)

```
GET    /v1/projects/{projectId}/devices/{deviceId}/stream   Query device stream
POST   /v1/projects/{projectId}/devices/{deviceId}/stream   Inject stream data (testing)
```

Query params for GET: `page`, `perPage`, `start` (RFC3339), `end` (RFC3339)

---

## Remote Procedure Call (RPC)

```
POST   /v1/projects/{projectId}/devices/{deviceId}/rpc
```

Body:
```json
{
  "method": "my_function",
  "params": [1, "arg2", true]
}
```

Query param: `timeout` (seconds, default 5). Response contains the return value or an error.

---

## Artifacts (Firmware Binaries)

Note: artifact CREATE uses the root path (no projectId), others are project-scoped.

```
POST   /v1/artifacts                                              Upload artifact (multipart/form-data)
GET    /v1/projects/{projectId}/artifacts                         List artifacts
GET    /v1/projects/{projectId}/artifacts/{artifactId}            Get artifact
PUT    /v1/projects/{projectId}/artifacts/{artifactId}            Update artifact metadata
PATCH  /v1/projects/{projectId}/artifacts/{artifactId}            Partial update
DELETE /v1/projects/{projectId}/artifacts/{artifactId}            Delete artifact
```

Upload artifact (multipart):
```bash
curl -X POST \
  -H "x-api-key: $GOLIOTH_API_KEY" \
  -F "projectId=$PROJECT_ID" \
  -F "version=1.2.0" \
  -F "blueprintId=$BLUEPRINT_ID" \
  -F "package=main" \
  -F "artifact=@path/to/firmware.bin" \
  "https://api.golioth.io/v1/artifacts"
```

List filter params: `version`, `blueprintId`, `package`, `ids`

---

## Blueprints (Device Templates)

```
GET    /v1/projects/{projectId}/blueprints                         List blueprints
POST   /v1/projects/{projectId}/blueprints                         Create blueprint
GET    /v1/projects/{projectId}/blueprints/{blueprintId}           Get blueprint
PUT    /v1/projects/{projectId}/blueprints/{blueprintId}           Replace blueprint
PATCH  /v1/projects/{projectId}/blueprints/{blueprintId}           Update blueprint
DELETE /v1/projects/{projectId}/blueprints/{blueprintId}           Delete blueprint
```

Create body:
```json
{
  "name": "nrf9160dk",
  "boardId": "optional-board-id",
  "platform": "zephyr"
}
```

---

## Packages (Firmware Component Names)

```
GET    /v1/organizations/{organizationId}/projects/{projectId}/packages                List packages
POST   /v1/organizations/{organizationId}/projects/{projectId}/packages                Create package
GET    /v1/organizations/{organizationId}/projects/{projectId}/packages/{packageId}    Get package
PUT    /v1/organizations/{organizationId}/projects/{projectId}/packages/{packageId}    Update package
DELETE /v1/organizations/{organizationId}/projects/{projectId}/packages/{packageId}    Delete package
```

---

## Cohorts (Device Targeting Groups)

```
GET    /v1/organizations/{organizationId}/projects/{projectId}/cohorts                 List cohorts
POST   /v1/organizations/{organizationId}/projects/{projectId}/cohorts                 Create cohort
GET    /v1/organizations/{organizationId}/projects/{projectId}/cohorts/{cohortId}      Get cohort
PUT    /v1/organizations/{organizationId}/projects/{projectId}/cohorts/{cohortId}      Update cohort
DELETE /v1/organizations/{organizationId}/projects/{projectId}/cohorts/{cohortId}      Delete cohort
```

Create body:
```json
{
  "name": "production-fleet",
  "deviceIds": ["device-id-1", "device-id-2"],
  "tagIds": ["tag-id"]
}
```

---

## Deployments (OTA Rollout)

```
GET    /v1/organizations/{organizationId}/projects/{projectId}/cohorts/{cohortId}/deployments                        List deployments
POST   /v1/organizations/{organizationId}/projects/{projectId}/cohorts/{cohortId}/deployments                        Create deployment
GET    /v1/organizations/{organizationId}/projects/{projectId}/cohorts/{cohortId}/deployments/{deploymentId}         Get deployment
```

Create deployment body — links a cohort to specific package versions:
```json
{
  "name": "v1.2.0-rollout",
  "packages": [
    {
      "packageId": "package-id",
      "artifactId": "artifact-id",
      "version": "1.2.0"
    }
  ]
}
```

---

## OTA Events & Monitoring

```
GET    /v1/organizations/{organizationId}/projects/{projectId}/ota-events         List OTA events
GET    /v1/organizations/{organizationId}/projects/{projectId}/ota-events/stats   OTA event statistics
```

Query params: `cohortId`, `packageId`, `deviceId`, `eventType`, `status`, `startTime`, `endTime`, `limit`, `pageToken`

`status` values: `pending`, `downloading`, `applying`, `completed`, `failed`

---

## Data Pipelines

```
GET    /v1/organizations/{organizationId}/projects/{projectId}/pipelines                   List pipelines
POST   /v1/organizations/{organizationId}/projects/{projectId}/pipelines                   Create pipeline
GET    /v1/organizations/{organizationId}/projects/{projectId}/pipelines/{pipelineId}      Get pipeline
PUT    /v1/organizations/{organizationId}/projects/{projectId}/pipelines/{pipelineId}      Update pipeline
DELETE /v1/organizations/{organizationId}/projects/{projectId}/pipelines/{pipelineId}      Delete pipeline
GET    /v1/organizations/{organizationId}/projects/{projectId}/pipelines/errors            List all pipeline errors
GET    /v1/organizations/{organizationId}/projects/{projectId}/pipelines/{pipelineId}/errors  Pipeline-specific errors
```

---

## API Keys

```
GET    /v1/projects/{projectId}/apikeys                    List API keys
POST   /v1/projects/{projectId}/apikeys                    Create API key
PATCH  /v1/projects/{projectId}/apikeys/{apikeyId}         Update API key
DELETE /v1/projects/{projectId}/apikeys/{apikeyId}         Delete API key
```

---

## Certificates & PKI

```
GET    /v1/projects/{projectId}/certificates                       List certificates
POST   /v1/projects/{projectId}/certificates                       Upload certificate
GET    /v1/projects/{projectId}/certificates/{certificateId}       Get certificate
DELETE /v1/projects/{projectId}/certificates/{certificateId}       Delete certificate

GET    /v1/organizations/{organizationId}/projects/{projectId}/pki/policies                         List PKI policies
POST   /v1/organizations/{organizationId}/projects/{projectId}/pki/policies                         Create PKI policy
GET    /v1/organizations/{organizationId}/projects/{projectId}/pki/policies/{policyId}              Get policy
PUT    /v1/organizations/{organizationId}/projects/{projectId}/pki/policies/{policyId}              Update policy
DELETE /v1/organizations/{organizationId}/projects/{projectId}/pki/policies/{policyId}              Delete policy

GET    /v1/organizations/{organizationId}/projects/{projectId}/pki/providers                        List PKI providers
POST   /v1/organizations/{organizationId}/projects/{projectId}/pki/providers                        Create provider
GET    /v1/organizations/{organizationId}/projects/{projectId}/pki/providers/{providerId}           Get provider
PUT    /v1/organizations/{organizationId}/projects/{projectId}/pki/providers/{providerId}           Update provider
DELETE /v1/organizations/{organizationId}/projects/{projectId}/pki/providers/{providerId}           Delete provider
GET    /v1/organizations/{organizationId}/projects/{projectId}/pki/providers/{providerId}/status    Provider status
```

---

## Boards & Integration Types (read-only reference data)

```
GET    /v1/boards             List supported hardware boards
GET    /v1/boards/{id}        Get board details
GET    /v1/integration-types  List available integration/pipeline types
```

---

## Usage & Quotas

```
GET    /v1/organizations/{organizationId}/quotas    Get quota limits
GET    /v1/organizations/{organizationId}/usage/*   Usage metrics
```
