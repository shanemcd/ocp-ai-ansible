# OpenShift AI Ansible Automation

Ansible automation for installing and configuring Red Hat OpenShift AI (RHOAI) with GPU support on OpenShift clusters.

## Features

- **Node Feature Discovery (NFD)**: Automatically detect hardware features
- **NVIDIA GPU Operator**: Manage GPU drivers and runtime
- **GCP GPU MachineSet**: Create GPU-enabled worker nodes on GCP
- **OpenShift AI**: Install and configure RHOAI with KServe in Standard deployment mode
- **Idempotent**: All playbooks can be run multiple times safely

## Prerequisites

### Cluster Requirements

- OpenShift 4.12 or later
- Cluster admin access
- At least 2 worker nodes
- For GPU support: GPU-capable worker nodes or ability to create them

### Local Requirements

- Python 3.9+
- Ansible 2.15+
- `kubernetes.core` Ansible collection
- `oc` CLI tool (optional, for manual verification)

## Quick Start

### 1. Install Dependencies

```bash
# Install Ansible collections
uv run ansible-galaxy collection install -r requirements.yml
```

### 2. Prepare Your Inventory

```bash
# Copy the example inventory
cp examples/inventory.yml.example inventory.yml

# Edit inventory.yml with your cluster details
vi inventory.yml
```

**Minimum required variables** to set in `inventory.yml`:
- `kubeconfig_path`: Path to your cluster's kubeconfig file

For GCP GPU nodes, also specify:
- `gpu_type`: GPU type (e.g., nvidia-tesla-t4)
- `machine_type`: Machine type (e.g., n1-standard-4)
- `gpu_machineset_replicas`: Number of GPU nodes (e.g., 1)

All other configuration (cluster name, GCP project, region, network, etc.) is auto-detected from your cluster!

### 3. Run the Installation

```bash
# Complete installation (NFD + GPU Operator + GPU MachineSet + OpenShift AI)
uv run ansible-playbook -i inventory.yml playbooks/site.yml

# Or run components individually:

# Step 1: Install Node Feature Discovery
uv run ansible-playbook -i inventory.yml playbooks/install-nfd.yml

# Step 2: Install NVIDIA GPU Operator
uv run ansible-playbook -i inventory.yml playbooks/install-nvidia-gpu.yml

# Step 3: Create GPU MachineSet (GCP only)
uv run ansible-playbook -i inventory.yml playbooks/create-gpu-machineset.yml

# Step 4: Install OpenShift AI
uv run ansible-playbook -i inventory.yml playbooks/install-openshift-ai.yml

# Step 5 (Optional): Deploy a model
# Uncomment model_* variables in inventory.yml first
uv run ansible-playbook -i inventory.yml playbooks/deploy-model.yml
```

## Playbooks

### `site.yml`
**Complete installation playbook** that runs all components in order:
1. Node Feature Discovery
2. NVIDIA GPU Operator
3. GPU MachineSet (GCP)
4. OpenShift AI

This is the recommended way to install the full stack.

### `install-nfd.yml`
Installs the Node Feature Discovery operator, which detects hardware features on cluster nodes.

### `install-nvidia-gpu.yml`
Installs both NFD and the NVIDIA GPU Operator. The GPU Operator manages GPU drivers and runtime components.

### `create-gpu-machineset.yml`
Creates a GPU-enabled MachineSet on GCP. This playbook:
- Auto-detects cluster configuration from existing MachineSets
- Creates a MachineSet with GPU nodes
- Waits for nodes to become Ready with GPU resources

### `install-openshift-ai.yml`
Installs OpenShift AI in Standard deployment mode (RawDeployment). This mode:
- Does not require Service Mesh or Serverless
- Simpler setup with fewer dependencies
- Good for single-tenant environments

### `deploy-model.yml`
Deploys a KServe model with vLLM runtime. This playbook:
- Creates a ServingRuntime with optimized vLLM configuration
- Deploys an InferenceService for the model
- Automatically configures resource limits and GPU tolerations
- Waits for the model to be ready

**Configuration**: Uncomment the `model_*` variables in your `inventory.yml` file (see the Model Deployment Configuration section in `examples/inventory.yml.example`).

**Key features**:
- Automatic context length adjustment to prevent OOM errors
- GPU memory utilization tuning
- Support for OCI and S3 model storage
- Optional authentication

## Architecture

### Standard Deployment Mode

This automation configures OpenShift AI in **Standard deployment mode** (RawDeployment):

- **DSCInitialization**: `serviceMesh.managementState: Removed`
- **DataScienceCluster**:
  - `kserve.defaultDeploymentMode: RawDeployment`
  - `serving.managementState: Removed`

**Benefits**:
- No Service Mesh or Serverless dependencies
- Simpler setup and maintenance
- Direct Kubernetes deployments

**Trade-offs**:
- No auto-scaling for model serving
- No advanced traffic management (canary deployments, traffic splitting)

### GPU Configuration

GPU nodes are created **without taints** by default, allowing OpenShift AI workloads to schedule on them. This is intentional to avoid resource constraints in small clusters.

## Configuration

### Required Inputs

**Kubeconfig**: Path to your OpenShift cluster kubeconfig
```yaml
kubeconfig_path: /path/to/your/kubeconfig
```

That's it! The cluster name and all GCP configuration is auto-detected.

### Automatic Configuration Detection

The `gcp_gpu_machineset` role automatically discovers cluster configuration from existing worker MachineSets via the Kubernetes API:

**Auto-detected values:**
- Infrastructure ID (cluster name)
- GCP project ID
- GCP region and zone
- VPC network and subnet names
- Service account email
- RHCOS image version
- Network tags

This means you don't need to provide cluster metadata files or manually configure GCP settings. The automation reads directly from:
```bash
oc get machineset -n openshift-machine-api
```

You can override any auto-detected value by setting it in your `inventory.yml`.

### Custom Variables

All roles support customization via variables. See:
- `roles/*/defaults/main.yml` for available variables
- `examples/inventory.yml.example` for common overrides

## Roles

### `node_feature_discovery`
Installs and configures the Node Feature Discovery operator.

**Key tasks**:
- Create NFD namespace and operator group
- Subscribe to NFD operator
- Create NodeFeatureDiscovery instance

### `nvidia_gpu_operator`
Installs and configures the NVIDIA GPU Operator.

**Key tasks**:
- Install NFD (dependency)
- Create GPU operator namespace and subscription
- Configure ClusterPolicy for GPU management
- Verify GPU resources are available

**Fixed issues**:
- Correct label selector for GPU driver daemonset
- Simplified GPU resource verification

### `gcp_gpu_machineset`
Creates GPU-enabled MachineSets on GCP with automatic configuration detection.

**Key features**:
- Auto-detects cluster configuration from existing worker MachineSets
- No metadata files or manual GCP configuration required
- Supports custom overrides for all auto-detected values
- Creates MachineSet matching cluster's network and image configuration
- Waits for GPU nodes to be Ready with GPU resources available

**Auto-detected configuration**:
- Infrastructure ID, GCP project/region/zone
- Network and subnet names
- Service account, RHCOS image, tags

**Fixed issues**:
- Removed `memoryMb` annotation to avoid conflicts with machine controller
- Removed taints from GPU nodes

### `openshift_ai`
Installs and configures OpenShift AI.

**Key tasks**:
- Create operator namespace and subscription
- Configure DSCInitialization for Standard mode
- Create DataScienceCluster with RawDeployment
- Verify KServe controller is running

**Configuration**:
- Standard deployment mode (no Service Mesh/Serverless)
- All major components enabled (dashboard, pipelines, model registry, etc.)

## Verification

After installation, verify the components:

```bash
# Check NFD
oc get pods -n openshift-nfd

# Check GPU Operator
oc get pods -n nvidia-gpu-operator
oc get nodes -o json | jq '.items[].status.capacity."nvidia.com/gpu"'

# Check GPU nodes
oc get machineset -n openshift-machine-api
oc get machines -n openshift-machine-api

# Check OpenShift AI
oc get pods -n redhat-ods-operator
oc get pods -n redhat-ods-applications
oc get datasciencecluster
oc get dscinitializ ation
```

## Troubleshooting

### GPU driver pods not starting

Check GPU node labels:
```bash
oc get nodes --show-labels | grep gpu
```

Verify GPU resources:
```bash
oc describe node <gpu-node-name> | grep -A 5 Capacity
```

### OpenShift AI pods pending

Check for resource constraints:
```bash
oc get pods -n redhat-ods-applications -o wide
oc describe pod <pod-name> -n redhat-ods-applications
```

### MachineSet not creating nodes

Check MachineSet status:
```bash
oc get machineset -n openshift-machine-api
oc describe machineset <machineset-name> -n openshift-machine-api
```

Check Machine status:
```bash
oc get machine -n openshift-machine-api
oc describe machine <machine-name> -n openshift-machine-api
```

## Contributing

Contributions are welcome! Please:

1. Test your changes on a real cluster
2. Ensure playbooks remain idempotent
3. Update documentation for any new variables or features
4. Follow existing code style and conventions

## License

Apache 2.0

## References

- [OpenShift AI Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed)
- [NVIDIA GPU Operator Documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Node Feature Discovery](https://kubernetes-sigs.github.io/node-feature-discovery/)
