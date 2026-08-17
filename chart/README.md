# Azahar Helm Chart

[![Version: 1.0.1](https://img.shields.io/badge/Version-1.0.1-informational?style=flat-square)](https://github.com/HenriqZimer/azahar-helm-chart)
[![AppVersion: latest](https://img.shields.io/badge/AppVersion-latest-informational?style=flat-square)](https://docs.linuxserver.io/images/docker-azahar/)

A Helm chart for [Azahar](https://docs.linuxserver.io/images/docker-azahar/) - the linuxserver.io
Nintendo 3DS emulator (based on Citra), served as a full desktop over the browser via KasmVNC.

## TL;DR

```bash
# Add the Helm repository
helm repo add azahar https://henriqzimer.github.io/azahar-helm-chart
helm repo update

# Install Azahar
helm install azahar azahar/azahar
```

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- A default `StorageClass` available in the cluster if you enable persistence (or your own
  `storageClass`/`existingClaim` values)
- Ingress controller (optional, only if `ingress.enabled: true`)
- cert-manager (optional, for automatic TLS certificates)
- A node exposing `/dev/dri` (`gpu.vendor: intel-amd`) or the [NVIDIA device plugin](https://github.com/NVIDIA/k8s-device-plugin)
  (`gpu.vendor: nvidia`) if you enable `gpu.enabled`

## Installing the Chart

### From Helm Repository

```bash
helm repo add azahar https://henriqzimer.github.io/azahar-helm-chart
helm repo update

helm install azahar azahar/azahar
```

### From Source

```bash
git clone https://github.com/HenriqZimer/azahar-helm-chart.git
cd azahar-helm-chart

helm package chart/
helm install azahar ./azahar-1.0.1.tgz
```

## Configuration

### Persistence

By default `persistence.config.enabled` is `false`, so `/config` (app settings, BIOS, save
states) is an ephemeral `emptyDir` — everything is lost when the pod restarts. For anything but a
quick test, enable it:

```yaml
persistence:
  config:
    enabled: true
    type: pvc # pvc | hostPath | emptyDir
    storageClass: "" # leave empty to use the cluster default
    size: 10Gi
```

### ROMs / BIOS library

The chart does not manage a ROMs volume directly. Use `extraVolumes`/`extraVolumeMounts` to
mount an existing PVC, NFS share, or hostPath:

```yaml
extraVolumes:
  - name: roms
    persistentVolumeClaim:
      claimName: my-roms-library
extraVolumeMounts:
  - name: roms
    mountPath: /roms
```

### Ingress

Disabled by default. Reach the UI via `kubectl port-forward` for quick testing, or:

```yaml
ingress:
  enabled: true
  className: "traefik"
  hosts:
    - host: azahar.example.com
      paths:
        - path: /
          pathType: Prefix
```

### Access control

The KasmVNC desktop has no authentication by default. Either restrict access at the network
layer (ingress auth, NetworkPolicy, VPN-only), or set:

```yaml
env:
  KASMVNC_ENABLE_BASIC_AUTH: "true"
  CUSTOM_USER: "myuser"
  PASSWORD: "changeme" # prefer envFrom with a Secret instead
```

### Azahar crashes when launching a game (SIGBUS)

If the dashboard/UI works but a game crashes on boot (check the container logs for
`Unhandled SIGBUS`), Azahar's JIT recompiler may need more `/dev/shm` than the small
default — this is confirmed for this chart's sibling, pcsx2-helm-chart (PCSX2's EE recompiler
uses a shared-memory-backed "fastmem" mmap trick and crashes with SIGBUS right after a game
starts when `/dev/shm` is too small); not independently confirmed on Azahar, but worth trying
first since the base image/toolchain is the same:

```yaml
shmSize: 2Gi # chart default; raise further if it still crashes
```

If that alone doesn't fix it, the JIT may also be hitting Docker/Kubernetes' default seccomp
profile (blocks syscalls it needs to allocate executable memory on some kernel/libseccomp
combinations):

```yaml
seccompUnconfined: true
```

### GPU / hardware acceleration

#### Intel/AMD (VA-API)

```yaml
gpu:
  enabled: true
  vendor: intel-amd
  devicePath: /dev/dri
  supplementalGroups: [44, 109] # host's video/render group GIDs; check with `getent group video render`
```

#### NVIDIA

Requires the [NVIDIA device plugin](https://github.com/NVIDIA/k8s-device-plugin) already installed on the
cluster (this chart only requests the resource, it doesn't install the plugin):

```yaml
gpu:
  enabled: true
  vendor: nvidia
  nvidia:
    count: 1
```

This sets `resources.limits."nvidia.com/gpu"` and the `NVIDIA_VISIBLE_DEVICES`/`NVIDIA_DRIVER_CAPABILITIES`
env vars — it does **not** need `devicePath`/`supplementalGroups`, the device plugin injects `/dev/nvidia*`
itself.

Two node-level prerequisites this chart cannot set for you (see the
[linuxserver.io GPU Configuration docs](https://docs.linuxserver.io/images/docker-azahar/#gpu-configuration)):

- The node needs `nvidia-drm.modeset=1 nvidia_drm.fbdev=1` on its kernel command line (e.g. via GRUB), or the
  compositor's own screen buffer allocation fails with `DRM_IOCTL_MODE_CREATE_DUMB failed: Permission denied`
  and the desktop renders solid black even though the container itself looks healthy.
- On a genuinely headless node (no monitor ever attached to that GPU), DRM may still need a physical HDMI/DP
  dummy plug in the port for the driver to initialize a display at all.

If Azahar itself crashes with `Unhandled SIGBUS` in the container logs on boot (i.e. the dashboard works
but launching a game doesn't), see `shmSize` below and `seccompUnconfined` above — try those first, it's
usually not a GPU problem.

### RomM emulator streaming — not available for Azahar

The sibling PCSX2/Dolphin/xemu/Eden charts (and, experimentally, RPCS3) can run as a target for
RomM's [Emulator Streaming](https://docs.romm.app/latest/using/emulator-streaming/) feature via a
third-party Docker Mod that adds an HTTP broker RomM talks to. **No such mod exists for Azahar** —
unlike PCSX2/Dolphin/xemu/Eden/RPCS3/PPSSPP, there isn't even a placeholder repository:
`azahar-romm-integration` does not exist anywhere on GitHub.

The chart still carries `streaming.enabled`/`streaming.brokerPort` in `values.yaml` so its shape
matches its siblings and it's ready to use if a mod ever ships, but there is nothing to enable
today — leave it `false` and use Azahar directly through its own desktop (`service.httpsPort`),
opening ROMs from wherever you mounted your library (see `extraVolumes` above).

## Values

| Key | Default | Description |
|---|---|---|
| `replicaCount` | `1` | Number of replicas. Must stay at 1 — no session clustering. |
| `image.repository` | `lscr.io/linuxserver/azahar` | Container image. |
| `image.tag` | `latest` | Image tag. |
| `env.PUID` / `env.PGID` | `1000` / `1000` | User/group the app runs as; match your storage's ownership. |
| `env.TZ` | `Etc/UTC` | Timezone. |
| `env.KASMVNC_ENABLE_BASIC_AUTH` | `false` | Require basic auth on the web UI. |
| `service.httpPort` / `service.httpsPort` | `3000` / `3001` | KasmVNC ports. |
| `streaming.enabled` | `false` | Expose the RomM streaming broker port (see above). |
| `streaming.brokerPort` | `8000` | Broker port to expose, must match the mod's `BROKER_PORT`. |
| `ingress.enabled` | `false` | Expose the HTTP port via Ingress. |
| `persistence.config.enabled` | `false` | Persist `/config`. `false` uses an ephemeral emptyDir. |
| `extraVolumes` / `extraVolumeMounts` | `[]` | Raw volume/volumeMount entries, e.g. a ROMs library. |
| `shmSize` | `2Gi` | Size of the memory-backed `/dev/shm` mount (KasmVNC and the JIT both need this). |
| `gpu.enabled` / `gpu.vendor` | `false` / `intel-amd` | Enable GPU passthrough; `intel-amd` (VA-API, `/dev/dri`) or `nvidia` (device plugin, `nvidia.com/gpu`). |
| `seccompUnconfined` | `false` | Disable the default seccomp profile if the JIT crashes with SIGBUS. |
| `resources` | `500m/1Gi` requests, `2/4Gi` limits | Container resources. |
| `serviceAccount.create` | `false` | Create a dedicated ServiceAccount. |

See [values.yaml](values.yaml) for the full, commented list.

## Uninstalling the Chart

```bash
helm uninstall azahar
```

This does not delete PVCs created by the chart; remove them manually if you no longer need the
data.

## License

MIT — see [LICENSE](../LICENSE).
