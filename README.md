# Golioth Management API Skill for Claude

A Claude Code skill that helps developers build applications on top of the [Golioth Management REST API](https://docs.golioth.io/management-api) — fleet dashboards, OTA rollout tools, pipeline configuration, and more.

Building a firmware update dashboard or need to automate device fleet management? Install the skill and describe what you want to build — it will guide you to the right endpoints, data models, and server-side patterns for your stack.

## Install

```bash
claude skill install https://github.com/goliothlabs/golioth-management-skill/raw/main/golioth-management-api.skill
```

Then start a new Claude Code session and describe what you're building.

## What it does

**Corrects hallucinations.** The Golioth Management API has a few sharp edges that LLMs consistently get wrong without guidance — fabricated "releases" entities, pipelines that silently fail because the YAML isn't base64-encoded, and WebSocket recommendations for a polling-only API. The skill prevents all of these.

**OTA rollouts and monitoring.** Covers the full entity model (artifact → package → deployment → cohort → device) and the staged rollout pattern: upload once, deploy the same artifact to a test cohort, then beta, then production, with monitoring at each gate using the OTA events endpoint.

**Pipeline management.** Explains how to create and update data pipelines via REST, including the base64-encoded YAML requirement, the enable/disable toggle pattern, and how to update a pipeline when device firmware switches from JSON to CBOR encoding.

**Node.js and TypeScript first.** Since the Management API is server-side only (CORS blocks browser requests), the skill provides TypeScript interfaces, a `golioth()` fetch helper, and complete Express and Next.js App Router examples.

## What it covers

- Authentication (`x-api-key`) and the two URL patterns (`/v1/projects/...` vs `/v1/organizations/.../projects/...`)
- Devices, tags, blueprints, and the LightDB State and Stream endpoints
- OTA: artifacts, packages, cohorts, deployments, and OTA events for per-device progress tracking
- Staged firmware rollout: test → beta → production with completion gates
- Firmware version dashboards: `device.metadata.update`, cohort grouping, out-of-date detection
- Pipelines: base64-encoded YAML, webhook/MongoDB/S3 destinations, CBOR→JSON transformer, troubleshooting via `/pipelines/errors`
- Node.js/TypeScript patterns — all API calls must be server-side
- Pagination, polling intervals, and rollback patterns

## Example prompts to try

- *"How do I do a staged firmware rollout to a test cohort, then a larger group, then the rest of the fleet? How do I monitor it throughout?"*
- *"I want to build a dashboard showing the current firmware version on each device and highlight which ones are out of date."*
- *"My devices just switched from JSON to CBOR encoding. How do I update my Golioth pipeline to handle this?"*
- *"How do I create a pipeline that sends device telemetry to a webhook, and how do I temporarily disable it without deleting it?"*
- *"Show me how to query the LightDB Stream for the last hour of temperature readings from my fleet using TypeScript."*

## Pair with the Firmware Skill

For end-to-end IoT development, combine this skill with the [Golioth Firmware Skill](https://github.com/goliothlabs/golioth-firmware-skill), which covers device-side development on Zephyr, ESP-IDF, NCS, and ModusToolbox. The firmware skill handles what runs on the device; this skill handles what you build on top of the cloud API.

## Resources

- [Golioth Management API docs](https://docs.golioth.io/management-api)
- [Golioth Console](https://console.golioth.io)
- [Golioth Firmware Skill](https://github.com/goliothlabs/golioth-firmware-skill) — device-side development
- [Golioth docs](https://docs.golioth.io)
