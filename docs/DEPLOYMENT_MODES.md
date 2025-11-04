# OpenShift AI Deployment Modes

OpenShift AI supports two deployment modes for KServe model serving. This automation uses **Standard deployment mode** by default.

## Standard Deployment Mode (RawDeployment)

### Overview

Standard mode deploys models as direct Kubernetes resources without requiring Service Mesh or Serverless components.

### Configuration

```yaml
# DSCInitialization
spec:
  serviceMesh:
    managementState: Removed

# DataScienceCluster
spec:
  components:
    kserve:
      managementState: Managed
      defaultDeploymentMode: RawDeployment
      rawDeploymentServiceConfig: Headed
      serving:
        managementState: Removed
```

### Benefits

- **Simpler Setup**: No Service Mesh or Serverless dependencies
- **Fewer Resources**: Lower resource requirements
- **Easier Troubleshooting**: Standard Kubernetes deployments and services
- **Good for**: Single-tenant environments, development, testing

### Limitations

- No auto-scaling for model serving
- No advanced traffic management (canary deployments, traffic splitting)
- No request-based scaling

### Components Installed

- KServe controller
- Model registry
- Data science pipelines
- Dashboard
- Workbenches
- Training operator
- Distributed workloads (CodeFlare, Kueue, Ray)

## Advanced Deployment Mode (Serverless)

### Overview

Advanced mode uses Knative Serving and Service Mesh for advanced model serving capabilities.

### Configuration

```yaml
# DSCInitialization
spec:
  serviceMesh:
    managementState: Managed
    controlPlane:
      name: data-science-smcp
      namespace: istio-system

# DataScienceCluster
spec:
  components:
    kserve:
      managementState: Managed
      defaultDeploymentMode: Serverless
      serving:
        managementState: Managed
        name: knative-serving
```

### Benefits

- **Auto-scaling**: Scale to zero and scale based on requests
- **Traffic Management**: Canary deployments, traffic splitting, A/B testing
- **Multi-tenancy**: Better isolation between tenants
- **Good for**: Production environments, multi-tenant setups

### Requirements

- Red Hat OpenShift Serverless operator
- Red Hat OpenShift Service Mesh operator
- Red Hat - Authorino operator (optional, for authorization)
- Additional resource overhead for Service Mesh components

### Limitations

- More complex setup and troubleshooting
- Higher resource requirements
- Requires understanding of Service Mesh concepts

## Switching Between Modes

### To Switch from Standard to Advanced

1. Install required operators:
   - Red Hat OpenShift Serverless
   - Red Hat OpenShift Service Mesh

2. Update DSCInitialization:
   ```bash
   oc patch dscinitializations default-dsci --type=merge -p='{
     "spec": {
       "serviceMesh": {
         "managementState": "Managed"
       }
     }
   }'
   ```

3. Update DataScienceCluster:
   ```bash
   oc patch datasciencecluster default-dsc --type=merge -p='{
     "spec": {
       "components": {
         "kserve": {
           "defaultDeploymentMode": "Serverless",
           "serving": {
             "managementState": "Managed"
           }
         }
       }
     }
   }'
   ```

### To Switch from Advanced to Standard

1. Update DataScienceCluster:
   ```bash
   oc patch datasciencecluster default-dsc --type=merge -p='{
     "spec": {
       "components": {
         "kserve": {
           "defaultDeploymentMode": "RawDeployment",
           "serving": {
             "managementState": "Removed"
           }
         }
       }
     }
   }'
   ```

2. Update DSCInitialization:
   ```bash
   oc patch dscinitializations default-dsci --type=merge -p='{
     "spec": {
       "serviceMesh": {
         "managementState": "Removed"
       }
     }
   }'
   ```

3. Optionally remove Service Mesh and Serverless operators if not needed

## Choosing the Right Mode

### Use Standard Mode When:

- Setting up development or test environments
- Working with single-tenant deployments
- Resource constraints are a concern
- Simplicity is preferred over advanced features
- Teams are not familiar with Service Mesh

### Use Advanced Mode When:

- Deploying in production with multiple tenants
- Auto-scaling is required
- Advanced traffic management is needed (canary, A/B testing)
- Teams have Service Mesh expertise
- Resources are not constrained

## References

- [OpenShift AI Documentation - KServe](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed)
- [Configuring automated installation of KServe](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed/2.13/html/serving_models/serving-large-models_serving-large-models#configuring-automated-installation-of-kserve_serving-large-models)
