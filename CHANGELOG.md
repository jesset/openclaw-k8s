# Changelog

## [0.2.0] - 2026-02-06

### Added
- CN-IM support: Feishu, DingTalk, QQ Bot, WeCom channels
- Optimized Dockerfile with multi-stage build (docker/)
- Bridge port support in Service (18790)
- CN-IM secret fields in Secret template
- values-minimal.yaml and values-production.yaml presets
- Health probes using `openclaw health` command
- **Config persistence mechanism**: `openclaw.forceReseedConfig` flag for controlled config updates
- **HostPath persistence option**: Support for `persistence.type=hostPath` for single-node clusters

### Changed
- Default image changed to `justlikemaki/openclaw-docker-cn-im`
- Container command changed from `node dist/index.js gateway` to `openclaw gateway --verbose`
- Container port now uses `service.targetPort` (unified port configuration)
- `readOnlyRootFilesystem` set to `false` (CN-IM image requires writable filesystem)
- Init container enhanced to sync extensions from image to PVC
- Chart version bumped to 0.2.0
- `appVersion` pinned to `"2026.2.3"` (was `latest`)
- `autoscaling.maxReplicas` default changed to `1` (was `3`)

### Fixed
- Init container first-run copy logic (mkdir moved after emptiness check)
- podAnnotations incorrectly applied to StatefulSet metadata (removed)
- Sidecars block placement (moved outside main container definition)
- NOTES.txt hardcoded port 18789 (now uses `{{ .Values.service.port }}`)
- Removed buggy unused `podLabels` helper

### Based on
- [openclaw-kubernetes](https://github.com/feiskyer/openclaw-kubernetes) v0.1.2
- [OpenClaw-Docker-CN-IM](https://github.com/justlovemaki/OpenClaw-Docker-CN-IM)
