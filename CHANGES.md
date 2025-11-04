# Recent Changes

## Auto-Detection of GCP Configuration

The `gcp_gpu_machineset` role has been updated to automatically detect all GCP cluster configuration from existing worker MachineSets via the Kubernetes API.

### What Changed

**Before:**
- Required a `metadata.json` file with cluster configuration
- Required manual specification of GCP project, region, zone, network, etc.
- Users had to extract and maintain cluster metadata separately

**After:**
- Automatically reads configuration from existing MachineSets
- No metadata files required
- Minimal configuration needed in inventory
- All values can still be overridden if needed

### Minimum Required Configuration

**For all playbooks:**
```yaml
kubeconfig_path: /path/to/kubeconfig
```

**For GPU MachineSet creation, also add:**
```yaml
gpu_type: nvidia-tesla-t4
machine_type: n1-standard-4
gpu_machineset_replicas: 1
```

That's it! No cluster name, project ID, region, or any other GCP configuration needed.

### Auto-Detected Values

The following are automatically detected from your cluster:
- Infrastructure ID (cluster name)
- GCP project ID
- GCP region and zone
- VPC network name
- Subnet name
- Service account email
- RHCOS image version
- Network tags

### Migration Guide

If you have existing inventory files with GCP configuration:

1. **Keep working** - Your existing configuration will continue to work as overrides
2. **Simplify** - You can remove GCP-specific variables and let them be auto-detected
3. **Override when needed** - Set specific values in inventory to override auto-detection

Example minimal inventory:
```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
  vars:
    kubeconfig_path: /path/to/kubeconfig
    gpu_type: nvidia-tesla-t4
    machine_type: n1-standard-4
    gpu_machineset_replicas: 1
```

### Benefits

1. **Simpler setup** - Less configuration required
2. **Fewer errors** - No need to manually extract cluster metadata
3. **Always up-to-date** - Uses current cluster configuration
4. **More portable** - Same automation works across different clusters
5. **Still flexible** - All values can be overridden when needed
