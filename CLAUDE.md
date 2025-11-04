# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Ansible automation for installing and configuring Red Hat OpenShift AI (RHOAI) with GPU support on OpenShift clusters running on GCP. The automation is fully idempotent and designed for Standard deployment mode (RawDeployment) without Service Mesh or Serverless dependencies.

## Common Commands

### Install Dependencies
```bash
# Install Ansible collections (uv will automatically install requirements.txt)
uv run ansible-galaxy collection install -r requirements.yml
```

### Running Playbooks

```bash
# Complete installation (recommended - runs all components in order)
uv run ansible-playbook -i inventory.yml playbooks/site.yml

# Individual component installation
uv run ansible-playbook -i inventory.yml playbooks/install-nfd.yml              # Node Feature Discovery
uv run ansible-playbook -i inventory.yml playbooks/install-nvidia-gpu.yml       # NFD + GPU Operator
uv run ansible-playbook -i inventory.yml playbooks/create-gpu-machineset.yml    # GPU nodes (GCP)
uv run ansible-playbook -i inventory.yml playbooks/install-openshift-ai.yml     # OpenShift AI

# Deploy a model with vLLM (uncomment model_* variables in inventory.yml first)
uv run ansible-playbook -i inventory.yml playbooks/deploy-model.yml             # KServe model deployment
```

### Verification Commands
```bash
# Check NFD
oc get pods -n openshift-nfd

# Check GPU Operator and resources
oc get pods -n nvidia-gpu-operator
oc get nodes -o json | jq '.items[].status.capacity."nvidia.com/gpu"'

# Check GPU MachineSets and nodes
oc get machineset -n openshift-machine-api
oc get machines -n openshift-machine-api

# Check OpenShift AI
oc get pods -n redhat-ods-operator
oc get pods -n redhat-ods-applications
oc get datasciencecluster
oc get dscinitializations
```

## High-Level Architecture

### Component Stack (Installation Order)
1. **Node Feature Discovery (NFD)** - Detects hardware features on cluster nodes
2. **NVIDIA GPU Operator** - Manages GPU drivers and runtime components (includes NFD as dependency)
3. **GPU MachineSet** (GCP only) - Creates GPU-enabled worker nodes
4. **OpenShift AI** - Installs RHOAI with KServe in Standard mode

### Deployment Philosophy

**Standard Deployment Mode (RawDeployment):**
- OpenShift AI is configured to use Standard deployment mode by default
- This mode does NOT require Service Mesh or Serverless operators
- KServe deployments use direct Kubernetes resources (Deployments/Services)
- Trade-offs: No auto-scaling or advanced traffic management, but simpler setup

**DSCInitialization configuration:**
```yaml
spec:
  serviceMesh:
    managementState: Removed  # No Service Mesh
```

**DataScienceCluster configuration:**
```yaml
spec:
  components:
    kserve:
      defaultDeploymentMode: RawDeployment  # Standard mode
      serving:
        managementState: Removed  # No Serverless
```

### Auto-Detection System

The `gcp_gpu_machineset` role automatically discovers cluster configuration from existing worker MachineSets via the Kubernetes API. This is a key architectural feature that eliminates the need for manual cluster metadata files.

**Auto-detected values:**
- Infrastructure ID (cluster name)
- GCP project ID, region, zone
- VPC network and subnet names
- Service account email
- RHCOS image version
- Network tags

**How it works:**
1. Query existing worker MachineSets: `oc get machineset -n openshift-machine-api`
2. Select first non-GPU worker MachineSet as reference
3. Extract `providerSpec.value` containing all GCP configuration
4. Allow override of any auto-detected value via inventory variables

**Minimal required configuration:**
```yaml
kubeconfig_path: /path/to/kubeconfig  # Required for all playbooks

# Additional for GPU MachineSet creation:
gpu_type: nvidia-tesla-t4
machine_type: n1-standard-4
gpu_machineset_replicas: 1
```

### Role Structure

Each role follows standard Ansible structure:
- `tasks/main.yml` - Main task execution
- `defaults/main.yml` - Default variable values
- `meta/main.yml` - Role metadata and dependencies
- `templates/` - Jinja2 templates (only in `gcp_gpu_machineset`)

**Role dependencies:**
- `nvidia_gpu_operator` depends on `node_feature_discovery`
- All other roles are independent

### GPU Configuration Details

**No Taints by Default:**
GPU nodes are created without taints to avoid resource constraints in small clusters. This allows OpenShift AI workloads to schedule freely on GPU nodes.

**GPU Driver Management:**
The ClusterPolicy configures:
- `defaultRuntime: crio` - Uses CRI-O runtime
- `use_ocp_driver_toolkit: true` - Uses OpenShift Driver Toolkit
- `kernelModuleType: auto` - Automatic kernel module detection
- DCGM exporter and service monitors enabled for GPU metrics

**Resource Verification:**
The automation waits for GPU resources to appear on nodes by checking:
1. GPU driver daemonset is ready (label: `app.kubernetes.io/component=nvidia-driver`)
2. Nodes have the label `feature.node.kubernetes.io/pci-10de.present=true` (NVIDIA PCI vendor ID)
3. Nodes report GPU capacity: `status.capacity."nvidia.com/gpu"`

### Idempotency Patterns

All playbooks can be run multiple times safely:
- Uses `kubernetes.core.k8s` module with `state: present` (creates or updates)
- Wait tasks use `until` with retries, not fixed sleeps
- MachineSet creation checks for existing resources before creating
- Operator subscriptions are upserted, not recreated

## Key Implementation Notes

### Template-Based MachineSet Creation
The `gcp_gpu_machineset` role uses a Jinja2 template (`templates/machineset.yml.j2`) to generate the MachineSet manifest with GPU configuration. The template combines auto-detected cluster config with user-specified GPU settings.

### OpenShift AI Component Management
The DataScienceCluster enables these components:
- `codeflare`, `kueue`, `ray` - Distributed workloads
- `dashboard` - Web UI
- `datasciencepipelines` - ML pipelines
- `kserve` - Model serving (RawDeployment mode)
- `modelmeshserving` - Multi-model serving
- `modelregistry` - Model versioning
- `trainingoperator` - Distributed training
- `trustyai` - Model bias detection
- `workbenches` - Jupyter notebooks

Explicitly disabled: `feastoperator`, `llamastackoperator`

### Wait Conditions and Timing
The automation uses specific wait conditions for each component:
- Operator deployments: Check `readyReplicas > 0` (30 retries × 10s)
- GPU driver daemonset: Check `numberReady > 0` (60 retries × 20s)
- GPU nodes: Check `Ready` condition + GPU capacity (60 retries × 20s)
- DataScienceCluster: Wait for `status.phase == "Ready"` (60 retries × 20s)
- KServe controller: Check pods in `Running` phase (60 retries × 20s)

## Common Gotchas

1. **MachineSet Creation**: Removed the `memoryMb` annotation from GPU MachineSet template to avoid conflicts with the machine controller, which manages memory settings automatically.

2. **GPU Driver Label Selector**: Uses `app.kubernetes.io/component=nvidia-driver` instead of a more generic selector to correctly identify GPU driver daemonsets.

3. **NFD Dependency**: The NVIDIA GPU Operator role includes NFD installation as a dependency. Running `install-nvidia-gpu.yml` installs both NFD and GPU Operator.

4. **DSCInitialization Wait**: Must wait for DSCInitialization to be created by the operator before patching it with Standard mode configuration.

5. **Inventory Setup**: Users should copy `examples/inventory.yml.example` to `inventory.yml` and customize it. The example uses `ansible_playbook_python` for the interpreter to ensure compatibility.
