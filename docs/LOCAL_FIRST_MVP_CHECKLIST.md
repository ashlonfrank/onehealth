# 1health.ai Local-First MVP Checklist

## Objective
Ship a stable local-first release of 1health.ai with strong privacy defaults, reliable data ingestion, and clear user documentation.

## Product Hardening
- [ ] Validate required environment variables on startup and show actionable errors.
- [ ] Improve sync error handling with explicit user-facing messages.
- [ ] Add source-specific import diagnostics for Whoop, Apple Health, Oura, and Fitbit.
- [ ] Add cache integrity checks and a documented cache reset workflow.

## Distribution
- [ ] Keep runtime local-first with no mandatory cloud backend.
- [ ] Publish self-hosted setup instructions for macOS and Linux.
- [ ] Define semantic versioning policy.
- [ ] Publish release notes template.

## Launch Docs
- [ ] Publish privacy model and data flow summary.
- [ ] Publish medical disclaimer and intended-use boundaries.
- [ ] Publish supported file formats and known parser limitations.

## Exit Criteria
- [ ] `npm run build` passes.
- [ ] `npm run test` passes.
- [ ] Rebrand sweep has no legacy name traces.
- [ ] README quick start and onboarding are complete.
