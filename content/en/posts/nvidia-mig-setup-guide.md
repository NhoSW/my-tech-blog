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

NVIDIA MIG (Multi-Instance GPU) fixes this. It splits a physical GPU into up to 7 independent instances so multiple workloads can share one card without stepping on each other. This post covers how we set up MIG on 4× A100s and wired it into Kubernetes.

---

## What Is MIG

MIG splits one physical GPU into several isolated instances. Each gets its own memory and compute units. They're fully independent — an OOM in one instance won't touch the others.

For the A100 80GB, the partition options look like this:

| Profile | Memory | SMs | Max Instances per GPU |
|---------|--------|-----|----------------------|
| 1g.10gb | ~10GB | 14 | 7 |
| 2g.20gb | ~20GB | 28 | 3 |
| 3g.40gb | ~40GB | 42 | 2 |
| 7g.80gb | ~80GB | 98 | 1 |

With 7-way partitioning (1g.10gb), four GPUs give you 28 instances. Simple math: 4x more GPU slots than before.

### Checking GPU Support

**MIG only works on certain GPUs.** A100 and H100 support it. A40 and V100 don't. We tried A40 first, hit the wall, and moved to A100.

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

The full list is in the [NVIDIA docs](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html#supported-gpus).

---

## Setting Up MIG

### Prerequisite: Disable nvidia-mig-manager

Before anything else, disable `nvidia-mig-manager.service`. This daemon turns MIG off on every boot, which will undo your work.

```bash
sudo systemctl disable nvidia-mig-manager.service
```

### Install mig-parted

`nvidia-mig-parted` is NVIDIA's tool for managing MIG configs. Write a YAML file describing what you want, run one command, done.

```bash
# Install deb package (Ubuntu/Debian)
curl -LO https://github.com/NVIDIA/mig-parted/releases/download/v0.5.2/nvidia-mig-manager_0.5.2-1_amd64.deb
sudo dpkg -i nvidia-mig-manager_0.5.2-1_amd64.deb
```

### Enable MIG Mode

```bash
sudo nvidia-smi -mig 1
```

This puts MIG mode in pending. To actually apply it, you need a **reboot.**

```bash
sudo reboot
```

> **VM caveat:** GPU passthrough VMs don't support `nvidia-smi --gpu-reset`. No reset means no MIG without a full reboot. We ended up moving from VMs to bare metal just for this.

### Writing config.yaml

mig-parted uses YAML to declare partition configs. Put multiple presets in one file and switch between them whenever you want.

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

The `custom-config` at the bottom is what matters most. It mixes different profiles per GPU. More on why later.

### Applying the Configuration

```bash
# Apply uniform 7-way split
sudo nvidia-mig-parted apply -f ./config.yaml -c all-1g.10gb

# Or apply custom config
sudo nvidia-mig-parted apply -f ./config.yaml -c custom-config
```

Add `-d` for debug logs. Once it's done, check the result with `nvidia-smi -L`:

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

MIG config resets on reboot. Fix this with a systemd service that reapplies it at boot.

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

We started simple. Split all 4 GPUs into 1g.10gb × 7, get 28 instances, let everyone share. What could go wrong?

### Large Models Don't Fit

A lot, it turns out. The recommendation team's models needed 30GB+ of GPU memory. Each MIG instance only had 10GB. CUDA OOM. Models wouldn't even load.

LLMs made it worse. Llama 2 7B failed on a MIG device even at 4-bit quantization. 13B? Not a chance.

**Your partition strategy has to match your workloads.** There's no way around it.

### Per-GPU Custom Partitioning

We looked at what each team actually ran and landed on this:

| GPU | Profile | Purpose |
|-----|---------|---------|
| GPU 0 | 3g.40gb × 2 | Mid-size model training (~40GB memory) |
| GPU 1 | 2g.20gb × 3 + 1g.10gb × 1 | Small-to-mid experiments |
| GPU 2 | 2g.20gb × 3 + 1g.10gb × 1 | Small-to-mid experiments |
| GPU 3 | 1g.10gb × 7 | Jupyter notebooks, light batch jobs |

If you need full GPU power for large model training, leave some GPUs with MIG off. We did exactly that as LLM experiments picked up.

> **Watch out:** Mixing MIG slices has physical constraints. Memory is allocated left-to-right inside the GPU and can't be shared vertically across slices. Check the [NVIDIA docs](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html) for which combos actually work.

---

## Kubernetes Integration

To use MIG instances in Kubernetes, you need to reconfigure the NVIDIA device plugin.

### MIG Strategies

The device plugin has three ways to expose MIG instances:

| Strategy | Resource Name | Description |
|----------|--------------|-------------|
| none | `nvidia.com/gpu` | MIG-unaware. Allocates whole physical GPUs |
| single | `nvidia.com/gpu` | Auto-assigns MIG instances. Existing workloads work as-is |
| mixed | `nvidia.com/mig-{profile}` | Exposes each MIG profile as a separate resource |

`single` is convenient. Existing workloads requesting `nvidia.com/gpu: 1` get a MIG instance without any code changes. The downside: you can't pick the size.

`mixed` gives you control. Request `nvidia.com/mig-3g.40gb: 1` and you get exactly that. If you're running different partitions per GPU, this is what you want.

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

Restart the nvidia-device-plugin pods after this. Then check `kubectl describe node` to see the new resources:

```
Allocatable:
  nvidia.com/mig-1g.10gb:  9
  nvidia.com/mig-2g.20gb:  6
  nvidia.com/mig-3g.40gb:  2
```

### Using MIG in JupyterHub

Switch the JupyterHub profile to request MIG resources instead of whole GPUs:

```yaml
# Before (whole GPU allocation)
extra_resource_limits:
  nvidia.com/gpu: "1"

# After (MIG instance allocation)
extra_resource_limits:
  nvidia.com/mig-1g.10gb: "1"
```

Heads up: with mixed strategy, `nvidia.com/gpu: 1` grabs an entire physical GPU. Always specify the MIG profile name.

### Airflow KubernetesPodOperator

Same idea for GPU tasks in Airflow. Swap the resource name:

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

### When to Reconfigure

Changing MIG config kills all processes on those GPUs. Pick a quiet time. Our batch jobs kicked off at 4 AM, so we did MIG changes after 9 PM.

### All-or-Nothing Per Server

You can't turn MIG on for some GPUs and leave it off for others on the same machine. **All GPUs must be MIG-enabled or all disabled.** Need a full GPU for large model training? Either use a different server or set `7g.80gb: 1` in your config to expose the whole GPU as one instance.

### Capacity Numbers Will Jump

Your Kubernetes GPU capacity looks very different after MIG. We went from 4 GPUs to 20+ allocatable instances. Update your monitoring dashboards and alert thresholds or you'll get false alarms.

---

## Conclusion

MIG works. It won't solve everything, but it makes expensive hardware go further. Here's what stuck with us:

1. **Check GPU support first.** A100 and H100 work. A40 and V100 don't. Save yourself the debugging.
2. **Skip VMs.** GPU passthrough can't do GPU reset. Bare metal makes life easier.
3. **Start uniform, go custom.** Begin with 7-way splits to get things running. Then tune per-GPU as you learn what your teams actually need.
4. **Big models need big GPUs.** LLMs and large recommendation models won't fit in MIG slices. Keep some full GPUs around.

GPUs are expensive. Slice them well and you get a lot more out of what you already have.

**References:**
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html)
- [nvidia-mig-parted (GitHub)](https://github.com/NVIDIA/mig-parted)
- [NVIDIA Device Plugin for Kubernetes](https://github.com/NVIDIA/k8s-device-plugin)
- [Getting the Most Out of the A100 GPU with MIG](https://developer.nvidia.com/blog/getting-the-most-out-of-the-a100-gpu-with-multi-instance-gpu/)
