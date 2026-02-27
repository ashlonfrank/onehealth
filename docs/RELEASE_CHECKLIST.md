# 1health.ai Release Checklist

## Pre-release
- [ ] Confirm package metadata (`name: 1health-ai`, version bumped).
- [ ] Confirm docs and UI use canonical `1health.ai` branding.
- [ ] Confirm no legacy branding references remain.
- [ ] Run `npm ci`.
- [ ] Run `npm run typecheck`.
- [ ] Run `npm run lint`.
- [ ] Run `npm run test`.
- [ ] Run `npm run build`.

## Privacy and Safety
- [ ] Verify no analytics/tracking dependencies were added.
- [ ] Verify local-only data handling remains unchanged.
- [ ] Verify disclaimer copy is present and accessible.

## Release
- [ ] Tag release with semantic version.
- [ ] Publish release notes with migration notes (if any).
- [ ] Announce known limitations and next milestone scope.
