# hami_kai

KAI scheduler + HAMi `kai-resource-isolator`, patched so that **GPU memory soft-quota is actually enforced** for KAI fractional-sharing pods.

Based on [Project-HAMi/KAI-resource-isolator](https://github.com/Project-HAMi/KAI-resource-isolator) (`update_v1.0`, Apache-2.0).

## Why this fork

Upstream `kai-resource-isolator` v1.0.0 injects `libvgpu.so` (HAMi-core) via `ld.so.preload` into pods that carry the KAI `gpu-memory` / `gpu-fraction` annotation, but it does **not** pass the per-pod memory limit to `libvgpu`. HAMi-core reads its cap from the `CUDA_DEVICE_MEMORY_LIMIT` env var, and the KAI binder never sets it — so `libvgpu` loads but enforces nothing (`nvidia-smi` shows the full device memory).

This patch makes the mutating webhook translate the KAI `gpu-memory` annotation (MiB) into
`CUDA_DEVICE_MEMORY_LIMIT=<value>m` on every container, so `libvgpu` enforces the requested cap.

### The change

`cmd/webhook/main.go` — when a pod has the `gpu-memory` annotation, the webhook adds a
`CUDA_DEVICE_MEMORY_LIMIT=<gpu-memory>m` env var to each (init)container (skipping containers
that already set it; handling the empty-env-array case). See `appendMemLimitEnvOp`.

`gpu-fraction` (compute share) carries no absolute memory value and is intentionally not handled.

## Build

The webhook binary is the only thing changed, so the fast path reuses the upstream image and
only swaps the binary (no need to rebuild `libvgpu.so`):

```bash
# docker/Dockerfile.memlimit
docker build -f docker/Dockerfile.memlimit -t <registry>/kai-resource-isolator:v1.0.0-memlimit .
docker push <registry>/kai-resource-isolator:v1.0.0-memlimit
```

To build the full image from scratch (rebuilding HAMi-core), use the upstream `docker/Dockerfile`
after `git submodule update --init --recursive`.

## Deploy (Helm, over the upstream chart)

```bash
helm upgrade --install kai-resource-isolator \
  oci://docker.io/projecthami/kai-resource-isolator --version 1.0.0-chart \
  -n kai-resource-isolator --create-namespace \
  --set image.repository=<registry>/kai-resource-isolator \
  --set image.tag=v1.0.0-memlimit \
  --set 'librarySync.nodeSelector.nvidia\.com/gpu\.present=true'
```

## Cluster prerequisites for KAI fractional GPU sharing

- KAI scheduler with `global.gpuSharing=true` and binder `--cdi-enabled=true` (KAI >= 0.14.x).
- The NVIDIA **`nvidia` container-runtime handler** must be registered (GPU Operator
  `cdi.nriPluginEnabled=false`) and `accept-nvidia-visible-devices-envvar-when-unprivileged=true`,
  because KAI passes the GPU to unprivileged shared pods via the `NVIDIA_VISIBLE_DEVICES` env
  (honored by the nvidia runtime, not by the NRI plugin for unprivileged pods).
- Shared pod spec: label `kai.scheduler/queue=<queue>`, annotation `gpu-memory: "<MiB>"`,
  `schedulerName: kai-scheduler`.

## Verify

```bash
# shared pod with annotation gpu-memory: "4000" -> nvidia-smi shows 0MiB / 4000MiB
kubectl logs <pod> | grep -E 'CUDA_DEVICE_MEMORY_LIMIT|MiB'
```
