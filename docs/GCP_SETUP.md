# GCP GPU Setup Guide

This guide covers creating GPU-enabled worker nodes on Google Cloud Platform (GCP) for OpenShift clusters.

## Prerequisites

- OpenShift cluster running on GCP
- Cluster admin access
- GCP quota for GPU instances in your region

## GPU Instance Types

### Available GPU Types on GCP

| GPU Type | Description | Use Case |
|----------|-------------|----------|
| `nvidia-tesla-t4` | 16GB VRAM, Turing architecture | Inference, light training |
| `nvidia-tesla-v100` | 16GB VRAM, Volta architecture | Training, inference |
| `nvidia-tesla-p4` | 8GB VRAM, Pascal architecture | Inference |
| `nvidia-tesla-p100` | 16GB VRAM, Pascal architecture | Training |
| `nvidia-tesla-k80` | 12GB VRAM (older) | Budget training |
| `nvidia-tesla-a100` | 40GB VRAM, Ampere architecture | High-performance training |

### Recommended Machine Types

| Machine Type | vCPUs | Memory | GPU Support |
|--------------|-------|---------|-------------|
| `n1-standard-4` | 4 | 15GB | 1-2 GPUs |
| `n1-standard-8` | 8 | 30GB | 1-4 GPUs |
| `n1-standard-16` | 16 | 60GB | 2-4 GPUs |
| `a2-highgpu-1g` | 12 | 85GB | 1x A100 |

## Configuration

### Inventory Variables

```yaml
# GCP project and location
gcp_project_id: my-gcp-project
gcp_region: us-east4
gcp_zone: us-east4-a

# Machine configuration
machine_type: n1-standard-4
gpu_type: nvidia-tesla-t4
gpu_count: 1

# Number of GPU nodes
gpu_machineset_replicas: 1

# Storage
root_volume_size: 128  # GB
root_volume_type: pd-ssd
```

### Auto-Discovery

The playbook automatically discovers most settings from your cluster:

```bash
# Infrastructure details
oc get infrastructure cluster -o json

# Existing MachineSet configuration
oc get machineset -n openshift-machine-api -o json
```

Auto-discovered values include:
- Infrastructure ID
- GCP project ID
- Region and zone
- Network and subnet names
- Service account email
- RHCOS image

You can override any auto-discovered value in your inventory.

## GPU Quota

Before creating GPU nodes, ensure you have quota in GCP:

```bash
# Check GPU quota
gcloud compute project-info describe --project=PROJECT_ID | grep -A 3 "NVIDIA"

# Request quota increase if needed
# Go to: https://console.cloud.google.com/iam-admin/quotas
```

## Node Taints

By default, this automation creates GPU nodes **without taints**. This allows all workloads to schedule on GPU nodes, which is appropriate for:

- Small clusters with limited resources
- Development and testing environments
- Dedicated GPU clusters

### Adding Taints (Optional)

If you want to reserve GPU nodes only for GPU workloads, you can add taints manually or modify the MachineSet template:

```yaml
# In roles/gcp_gpu_machineset/templates/machineset.yml.j2
spec:
  taints:
    - key: nvidia.com/gpu
      value: "true"
      effect: NoSchedule
```

Then pods need tolerations:

```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

## Verification

After creating GPU nodes:

```bash
# Check MachineSet
oc get machineset -n openshift-machine-api

# Check Machines
oc get machine -n openshift-machine-api

# Check Nodes
oc get nodes -l node-role.kubernetes.io/gpu

# Check GPU resources
oc get nodes -o json | jq '.items[].status.capacity."nvidia.com/gpu"'

# Check GPU labels
oc get nodes --show-labels | grep nvidia

# Verify GPU driver
oc debug node/<gpu-node-name> -- chroot /host nvidia-smi
```

## Troubleshooting

### Machine stuck in Provisioning

```bash
# Check Machine status
oc describe machine <machine-name> -n openshift-machine-api

# Check GCP console for VM creation
# https://console.cloud.google.com/compute/instances

# Common issues:
# - Insufficient GPU quota
# - GPU type not available in zone
# - Network/firewall issues
```

### GPU not detected

```bash
# Check NFD labels
oc get nodes -l feature.node.kubernetes.io/pci-10de.present=true

# Check GPU operator pods
oc get pods -n nvidia-gpu-operator

# Check GPU driver logs
oc logs -n nvidia-gpu-operator -l app=nvidia-driver-daemonset
```

### Node not joining cluster

```bash
# Check Machine events
oc get events -n openshift-machine-api --field-selector involvedObject.name=<machine-name>

# Check cloud-init logs on node
oc debug node/<node-name> -- chroot /host journalctl -u cloud-init
```

## Cost Optimization

### Preemptible GPU Nodes

GCP offers preemptible instances at lower cost. To use preemptible GPU nodes, modify the MachineSet template:

```yaml
# Add to providerSpec.value
preemptible: true
```

**Note**: Preemptible instances can be terminated by GCP at any time, so only use for fault-tolerant workloads.

### GPU Scheduling

To maximize GPU utilization:

1. **Time-slicing**: Allow multiple pods to share a GPU
2. **Multi-Instance GPU (MIG)**: Partition A100 GPUs into smaller instances
3. **Node autoscaling**: Scale GPU nodes based on demand

## Regional Considerations

### GPU Availability by Region

Not all GPU types are available in all regions. Check availability:

```bash
gcloud compute accelerator-types list --filter="zone:(us-east4-a)"
```

Common regions with good GPU availability:
- `us-central1` (Iowa)
- `us-east4` (Northern Virginia)
- `us-west1` (Oregon)
- `europe-west4` (Netherlands)

## References

- [GCP GPU Platforms](https://cloud.google.com/compute/docs/gpus)
- [GCP Machine Types](https://cloud.google.com/compute/docs/machine-types)
- [OpenShift Machine API](https://docs.openshift.com/container-platform/latest/machine_management/index.html)
