---
title: "NVIDIA MIG Setup Guide: Splitting One A100 GPU Into Many"
date: 2026-02-23
draft: false
categories: [Data Engineering]
tags: [nvidia, gpu, mig, kubernetes, infrastructure]
showTableOfContents: true
summary: "Four A100 GPUs can yield up to 28 independent GPU instances. This guide covers MIG concepts, mig-parted installation, diverse slice configurations, and Kubernetes integration based on hands-on experience."
---

GPUs are expensive. A single A100 costs tens of thousands of dollars, yet most workloads don't use all 80GB of memory. Allocating an entire GPU for a simple Jupyter notebook experiment means 70GB sits idle.

NVIDIA MIG (Multi-Instance GPU) solves this problem. It partitions a single physical GPU into up to 7 independent instances, allowing multiple workloads to run simultaneously. This post documents the process of setting up MIG on a 4× A100 environment and integrating it with Kubernetes.

---

## What Is MIG

MIG splits a single physical GPU into multiple isolated GPU instances. Each instance has its own dedicated memory and compute units, so they don't interfere with each other. An OOM crash in one instance doesn't affect the others.

Here are the partition options for the A100 80GB:

| Profile | Memory | SMs | Max Instances per GPU |
|---------|--------|-----|----------------------|
| 1g.10gb | ~10GB | 14 | 7 |
| 2g.20gb | ~20GB | 28 | 3 |
| 3g.40gb | ~40GB | 42 | 2 |
| 7g.80gb | ~80GB | 98 | 1 |

With 7-way partitioning (1g.10gb), four GPUs yield 28 instances. That's a 4x+ improvement in GPU utilization by simple math.

### Checking GPU Support

**MIG is only supported on specific GPUs like the A100 and H100.** The A40 and V100 do not support MIG. We initially tried on an A40 server, confirmed it wasn't supported, and switched to A100.

Check the `MIG M.` column in the `nvidia-smi` output:

```
# GPU without MIG support (A40)
| MIG M. |
|  N/A   |   ← Not supported

# GPU with MIG support (A100)
| MIG M.   |
| Disabled |   ← Supported but inactive
| Enabled  |   ← Active
```

See the [NVIDIA documentation](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html#supported-gpus) for the full list of supported GPUs.

---

## Setting Up MIG

### Prerequisite: Disable nvidia-mig-manager

Before enabling MIG, you must disable `nvidia-mig-manager.service`. This daemon resets MIG to disabled on every boot.

```bash
sudo systemctl disable nvidia-mig-manager.service
```

### Install mig-parted

`nvidia-mig-parted` is NVIDIA's MIG configuration tool. You write a declarative config file and it applies the desired partition layout in one shot.

```bash
# Install deb package (Ubuntu/Debian)
curl -LO https://github.com/NVIDIA/mig-parted/releases/download/v0.5.2/nvidia-mig-manager_0.5.2-1_amd64.deb
sudo dpkg -i nvidia-mig-manager_0.5.2-1_amd64.deb
```

### Enable MIG Mode

```bash
sudo nvidia-smi -mig 1
```

This puts MIG mode in a pending state. A **reboot is required** to apply it.

```bash
sudo reboot
```

> **VM caveat:** Virtual machines using GPU passthrough do not support `nvidia-smi --gpu-reset`. You cannot enable MIG without a reboot, so bare-metal servers are strongly recommended. We had to migrate from VMs to bare metal because of this issue.

### Writing config.yaml

mig-parted uses a YAML file to declare partition configurations. You can define multiple presets in a single file and switch between them as needed.

```yaml
version: v1
mig-configs:
  # Disable MIG
  all-disabled:
    - devices: all
      mig-enabled: false

  # 7-way split on all GPUs (1g.10gb × 7)
  all-1g.10gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "1g.10gb": 7

  # 3-way split on all GPUs (2g.20gb × 3)
  all-2g.20gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "2g.20gb": 3

  # 2-way split on all GPUs (3g.40gb × 2)
  all-3g.40gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "3g.40gb": 2

  # Full GPU as single instance (7g.80gb × 1)
  all-7g.80gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "7g.80gb": 1

  # Per-GPU custom configuration
  custom-config:
    - devices: [0]
      mig-enabled: true
      mig-devices:
        "3g.40gb": 2
    - devices: [1]
      mig-enabled: true
      mig-devices:
        "2g.20gb": 3
        "1g.10gb": 1
    - devices: [2]
      mig-enabled: true
      mig-devices:
        "2g.20gb": 3
        "1g.10gb": 1
    - devices: [3]
      mig-enabled: true
      mig-devices:
        "1g.10gb": 7
```

The `custom-config` at the bottom is the key part — it mixes different profiles per GPU. Why this matters is explained later.

### Applying the Configuration

```bash
# Apply uniform 7-way split
sudo nvidia-mig-parted apply -f ./config.yaml -c all-1g.10gb

# Or apply custom config
sudo nvidia-mig-parted apply -f ./config.yaml -c custom-config
```

Add the `-d` flag for debug logs. After applying, verify the instances with `nvidia-smi -L`:

```
GPU 0: NVIDIA A100-SXM4-80GB (UUID: GPU-xxxx)
  MIG 3g.40gb     Device  0: (UUID: MIG-xxxx)
  MIG 3g.40gb     Device  1: (UUID: MIG-xxxx)
GPU 1: NVIDIA A100-SXM4-80GB (UUID: GPU-xxxx)
  MIG 2g.20gb     Device  0: (UUID: MIG-xxxx)
  MIG 2g.20gb     Device  1: (UUID: MIG-xxxx)
  MIG 2g.20gb     Device  2: (UUID: MIG-xxxx)
  MIG 1g.10gb     Device  3: (UUID: MIG-xxxx)
...
```

### Persisting Across Reboots

MIG configuration resets on reboot. Register a systemd service to reapply it automatically.

```ini
# /etc/systemd/system/nvidia-mig-config.service
[Unit]
Description=Apply NVIDIA MIG Configuration
After=nvidia-persistenced.service

[Service]
Type=oneshot
ExecStart=/usr/bin/nvidia-mig-parted apply -f /home/user/config.yaml -c custom-config
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable nvidia-mig-config.service
```

---

## Lessons Learned: Why Uniform Partitioning Falls Short

Our first approach was simple: split all 4 GPUs into 1g.10gb × 7 for 28 total instances. Maximum concurrency — what could go wrong?

### Large Models Don't Fit in MIG Instances

Reality hit quickly. The recommendation team's models needed 30GB+ of GPU memory. A single MIG instance only had 10GB, so models failed to load with CUDA OOM errors.

LLMs were even worse. Llama 2 7B couldn't load on a MIG device even with 4-bit quantization. The 13B model was out of the question.

The conclusion was clear: **partition strategy must match the workload.**

### Per-GPU Custom Partitioning

After analyzing team workloads, we settled on this configuration:

| GPU | Profile | Purpose |
|-----|---------|---------|
| GPU 0 | 3g.40gb × 2 | Mid-size model training (~40GB memory) |
| GPU 1 | 2g.20gb × 3 + 1g.10gb × 1 | Small-to-mid experiments |
| GPU 2 | 2g.20gb × 3 + 1g.10gb × 1 | Small-to-mid experiments |
| GPU 3 | 1g.10gb × 7 | Jupyter notebooks, light batch jobs |

For large model training, you can also keep some GPUs with MIG disabled entirely. We ended up disabling MIG on some GPUs as LLM experimentation grew.

> **Note:** There are physical constraints when mixing MIG slices. Memory slices within a GPU are allocated left-to-right and cannot be shared vertically. Always check the supported combinations in the [NVIDIA documentation](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html).

---

## Kubernetes Integration

Using MIG instances in Kubernetes requires changes to the NVIDIA device plugin.

### Understanding MIG Strategies

The device plugin supports three MIG strategies:

| Strategy | Resource Name | Description |
|----------|--------------|-------------|
| none | `nvidia.com/gpu` | MIG-unaware. Allocates whole physical GPUs |
| single | `nvidia.com/gpu` | Auto-assigns MIG instances. Existing workloads work as-is |
| mixed | `nvidia.com/mig-{profile}` | Exposes each MIG profile as a separate resource |

The `single` strategy maps existing `nvidia.com/gpu: 1` requests to MIG instances without code changes. Convenient, but you can't control which instance size you get.

The `mixed` strategy lets you request specific profiles like `nvidia.com/mig-3g.40gb: 1`. This is the right choice when using different partitions per GPU.

### Updating the Device Plugin

```yaml
# values.yaml
migStrategy: mixed
```

```bash
helm upgrade -i nvdp nvdp/nvidia-device-plugin \
  --namespace nvidia-device-plugin \
  --create-namespace \
  --version 0.12.3 \
  -f ./values.yaml
```

After applying, restart the nvidia-device-plugin pods to refresh node resources. Verify with `kubectl describe node`:

```
Allocatable:
  nvidia.com/mig-1g.10gb:  9
  nvidia.com/mig-2g.20gb:  6
  nvidia.com/mig-3g.40gb:  2
```

### Using MIG in JupyterHub

Update the JupyterHub profile to request MIG-typed resources:

```yaml
# Before (whole GPU allocation)
extra_resource_limits:
  nvidia.com/gpu: "1"

# After (MIG instance allocation)
extra_resource_limits:
  nvidia.com/mig-1g.10gb: "1"
```

Be careful: with the mixed strategy, requesting `nvidia.com/gpu: 1` allocates an entire physical GPU. You must specify the MIG profile name to get an instance.

### Using MIG in Airflow KubernetesPodOperator

For GPU tasks in Airflow, update the resource requests to MIG types as well:

```python
resources = {
    "cpu": "4",
    "memory": "16Gi",
    "nvidia.com/mig-2g.20gb": "1",
}

KubernetesPodOperator(
    ...
    container_resources=k8s.V1ResourceRequirements(
        requests=resources,
        limits=resources,
    ),
    ...
)
```

---

## Operational Tips

### Timing MIG Reconfiguration

Changing MIG configuration requires terminating all processes on the affected GPUs. In production, schedule changes during idle windows. Our batch jobs started at 4 AM, so we ran MIG reconfiguration after 9 PM.

### MIG Must Be Enabled on All GPUs in a Server

You cannot enable MIG on some GPUs while leaving others disabled on the same server. **All GPUs must be either MIG-enabled or all MIG-disabled.** If you need a non-MIG GPU for large model training, either use a separate server or configure `7g.80gb: 1` in your MIG config to expose the full GPU as a single instance.

### GPU Capacity Changes

MIG dramatically changes your Kubernetes cluster's GPU capacity numbers. In our environment, 4 physical GPUs (capacity: 4) became 20+ instances after MIG. Adjust your monitoring and alerting thresholds accordingly.

---

## Conclusion

MIG is a great way to improve GPU utilization, but it's not a silver bullet. Here's what we learned:

1. **Check GPU support first.** Only specific models like A100 and H100 support MIG. A40 and V100 do not.
2. **Avoid VM environments.** GPU passthrough doesn't support GPU reset, making MIG activation painful. Bare metal is the way to go.
3. **Start uniform, evolve to custom.** Analyze workloads and mix different profiles per GPU for practical results.
4. **Large models don't fit in MIG.** LLMs and large recommendation models need full GPU memory. Keep some non-MIG GPUs available.

GPUs are expensive, but slice them right and you can do a lot more with the hardware you already have.

**References:**
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html)
- [nvidia-mig-parted (GitHub)](https://github.com/NVIDIA/mig-parted)
- [NVIDIA Device Plugin for Kubernetes](https://github.com/NVIDIA/k8s-device-plugin)
- [Getting the Most Out of the A100 GPU with MIG](https://developer.nvidia.com/blog/getting-the-most-out-of-the-a100-gpu-with-multi-instance-gpu/)
