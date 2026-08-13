# Changelog

All notable changes to this Helm chart will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.0.0] - 2026-08-13

### Added
- Initial release of the Azahar Helm chart
- Deployment, Service, optional Ingress and PVC for the linuxserver.io Azahar KasmVNC webtop image
- Configurable `serviceAccount` (create/name/annotations)
- Readiness and liveness probes (TCP check on the KasmVNC HTTP port)
- GPU passthrough for Intel/AMD (VA-API via `/dev/dri`) and NVIDIA (via the NVIDIA Kubernetes
  device plugin, `gpu.vendor: nvidia`)
- `seccompUnconfined` and a `2Gi` default `shmSize`, carried over from this chart's sibling
  pcsx2-helm-chart after PCSX2's JIT recompiler was found to crash with `SIGBUS` under Docker/
  Kubernetes' default seccomp profile / small `/dev/shm` — not independently confirmed on Azahar,
  but the same base image/toolchain makes it a likely fix if you hit the same crash
- `extraVolumes`/`extraVolumeMounts` for mounting an existing ROMs/firmware library
- `streaming.enabled`/`streaming.brokerPort` in `values.yaml`, kept only so the chart's shape
  matches its siblings. **No RomM broker mod exists for Azahar at all** — unlike the other
  sibling charts, there isn't even a placeholder repo; `azahar-romm-integration` does not exist
  anywhere on GitHub — there is nothing to enable yet.
