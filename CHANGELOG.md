# Changelog

All notable changes to this Helm chart will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.4.2] - 2026-08-23

### Changed
- `readinessProbe.initialDelaySeconds` 10s->30s, `livenessProbe.initialDelaySeconds` 30s->90s, and `resources.requests.cpu` 500m->1 - confirmed in production that under CPU contention on a shared GPU node, the old tight timing killed containers mid-boot (KasmVNC desktop hadn't finished starting) causing a self-sustaining restart storm; the higher CPU request also gives the scheduler an honest signal instead of letting far more pods pack onto a node than it can actually sustain under load.

## [1.3.0] - 2026-08-21

### Added
- `probes.enabled` (default `true`). Set to `false` to disable the readiness/liveness probe and the `http`/`https` `containerPort` declarations. Needed when running two `sunshine.hostNetwork` pods on the same node: the Selkies nginx (3000/3001) is hardcoded and can't be moved, so the second pod's probe/hostPort would otherwise crashloop it or block scheduling entirely even though Sunshine itself works fine. Tradeoff: the web UI (KasmVNC) becomes unreachable on that pod - Moonlight is unaffected.

## [1.2.1] - 2026-08-21

### Fixed
- `sunshine.*` now also mounts `/dev/uhid` - without it, Sunshine falls back to emulating gamepads as a generic Xbox One controller when the client reports a PlayStation-type pad (DualShock/DualSense), and that fallback does not correctly forward face buttons/triggers (confirmed: analog stick axes work, buttons don't). The real virtual DualShock/DualSense device needs kernel `uhid` access to be created.

## [1.2.0] - 2026-08-21

### Changed
- **BREAKING**: `sunshine.ports.*` (8 independent fields) replaced by a single `sunshine.port` (default `47989`). Sunshine only accepts one configurable base port ("port" in `sunshine.conf`) - every other port it listens on is a fixed offset from it. The old `ports.*` fields only ever affected the Deployment's `containerPort` entries - Sunshine itself always listened on its hardcoded defaults regardless. Also required a matching `azahar-sunshine-mod` update to write `port = $SUNSHINE_PORT` into `sunshine.conf` - confirmed broken end-to-end on the real cluster (both azahar and dolphin needing hostNetwork on the same GPU node) before this fix.

## [1.1.0] - 2026-08-20

### Added
- `sunshine.*` (POC) - runs Sunshine alongside the default Selkies/KasmVNC streaming for lower-latency Moonlight access, requiring a Docker Mod that installs Sunshine (via the LizardByte pacman repo, since this image is Arch Linux) and swaps the base image's Xvfb (not linked against libudev, so it never hotplugs Sunshine's uinput-created input devices) for a real Xorg + "dummy" driver. Needs `hostNetwork: true` (Moonlight is raw TCP/UDP, can't go through the Traefik Ingress, and the pod needs the host's network namespace for uinput hotplug uevents to reach udev) and `privileged: true` (no native Kubernetes knob for the device cgroup rule the pod needs to open `/dev/input/eventN` nodes created at runtime). Ported from dolphin-sunshine-mod - see that mod's README for the full list of issues found and fixed getting input to work, plus this image's Arch-specific quirks (pacman instead of apt, and an Xorg build that refuses `-config <path>` under privilege elevation).

## [1.0.5] - 2026-08-17

### Fixed
- README install commands now use a unique repo alias (`azahar-helm-chart`) instead of the bare chart name, and pin `--version` explicitly.

## [1.0.4] - 2026-08-17

### Changed
- README now shows the emulator's logo at the top.

## [1.0.3] - 2026-08-17

### Added
- Chart releases are now GPG-signed (`helm package --sign`) - see `artifacthub.io/signKey` in `Chart.yaml` for the public key URL and fingerprint. Powers the "Signed" badge on ArtifactHub.

## [1.0.2] - 2026-08-16

### Added
- `chart/values.schema.json` validating `values.yaml` - powers the "Values schema" feature on ArtifactHub, previously absent since no chart in this project ever had one.

## [1.0.1] - 2026-08-16

### Added
- `icon` in Chart.yaml, pointing at linuxserver.io's own logo image for this app - was missing entirely before, which is why no image ever showed up on ArtifactHub for this chart.

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
