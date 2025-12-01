# OpenShift Ansible Automation Platform GitOps

GitOps-based deployment of Ansible Automation Platform (AAP) 2.6 on OpenShift Container Platform using Argo CD.

## Overview

This repository provides a declarative, GitOps-native approach to deploying and managing Ansible Automation Platform 2.6 on OpenShift. It uses Argo CD for continuous deployment and Kustomize for manifest management.

**Key Features:**
- Automated deployment of AAP Operator and Platform instance
- Leverages Red Hat Community of Practice (CoP) validated configurations
- Self-healing and auto-sync capabilities
- Proper dependency management between operator and instance deployment
- RBAC configuration for Argo CD automation

## Repository Structure

```
ocp-ansible-gitops/
|-- bootstrap/
|   `-- root-app.yaml           # Argo CD Application manifests
|-- components/
|   |-- aap-operator/           # AAP Operator deployment
|   |   |-- kustomization.yaml  # Operator Kustomize config
|   |   `-- rbac.yaml           # RBAC for Argo CD
|   `-- aap-instance/           # AAP Platform instance
|       |-- kustomization.yaml  # Instance Kustomize config
|       `-- platform.yaml       # AAP CR configuration
`-- README.md
```

## Components

### AAP Operator (`components/aap-operator`)

Deploys the Ansible Automation Platform Operator using manifests from the [Red Hat CoP GitOps Catalog](https://github.com/redhat-cop/gitops-catalog). The operator manages the lifecycle of AAP platform components.

**Includes:**
- AAP Operator subscription (stable-2.6 channel)
- RBAC for Argo CD service account to manage AAP resources

### AAP Platform Instance (`components/aap-instance`)

Deploys the full AAP 2.6 platform with the following components:

- **Automation Gateway** - New in AAP 2.6, serves as the unified API gateway
- **Automation Controller** - Standard AAP UI and API (formerly Tower)
- **Automation Hub** - Container registry and content collections repository
- **PostgreSQL Database** - Operator-managed database for AAP components

## Prerequisites

### Required Access

1. **OpenShift Cluster** with cluster-admin privileges
2. **OpenShift GitOps Operator** installed (provides Argo CD)
3. **Network Access** to:
   - This Git repository
   - Red Hat CoP GitOps Catalog repository
   - Container image registries (registry.redhat.io, quay.io)

### Required Credentials

- Valid Red Hat subscription with AAP entitlements
- Pull secret configured in OpenShift for registry.redhat.io

## Argo CD Configuration

### Step 1: Install OpenShift GitOps Operator

If not already installed:

```bash
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Wait for the operator to be ready:

```bash
oc wait --for=condition=Ready pod -l name=openshift-gitops-operator -n openshift-operators --timeout=300s
```

### Step 2: Fork or Clone This Repository

If using a private repository, configure Argo CD with repository credentials:

```bash
# Create a secret with repository credentials
oc create secret generic repo-credentials \
  -n openshift-gitops \
  --from-literal=username=<git-username> \
  --from-literal=password=<git-token>

# Label the secret for Argo CD
oc label secret repo-credentials \
  -n openshift-gitops \
  argocd.argoproj.io/secret-type=repository
```

### Step 3: Deploy the Bootstrap Application

Apply the root Argo CD Application manifest:

```bash
oc apply -f bootstrap/root-app.yaml
```

This creates two Argo CD Applications:
1. **aap-bootstrap** - Deploys the AAP Operator
2. **aap-instance** - Deploys the AAP Platform instance

### Step 4: Verify Deployment

Monitor the Argo CD Applications:

```bash
# Check Application status
oc get applications -n openshift-gitops

# Watch operator installation
oc get csv -n ansible-automation-platform

# Monitor AAP platform deployment
oc get ansibleautomationplatform -n ansible-automation-platform
```

Access the Argo CD UI:

```bash
# Get the Argo CD route
oc get route openshift-gitops-server -n openshift-gitops

# Get admin password
oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d
```

## Configuration

### Argo CD Application Settings

The applications are configured with the following sync policies:

```yaml
syncPolicy:
  automated:
    prune: true        # Delete resources removed from Git
    selfHeal: true     # Correct drift automatically
  syncOptions:
    - CreateNamespace=true  # Auto-create target namespace
```

**Key Settings:**
- **Source Repository**: https://github.com/SiriXAU/ocp-ansible-gitops.git
- **Target Revision**: `main` branch
- **Destination Namespace**: `ansible-automation-platform`
- **Argo CD Namespace**: `openshift-gitops`

### Customizing the AAP Instance

Edit `components/aap-instance/platform.yaml` to customize:

**Automation Controller:**
```yaml
automation_controller_spec:
  replicas: 1                    # Number of controller replicas
  admin_user: admin              # Admin username
  loadbalancer_port: 80          # Service port
  projects_storage_size: 8Gi     # Persistent storage for projects
```

**Automation Hub:**
```yaml
automation_hub_spec:
  replicas: 1                    # Number of hub replicas
  postgres_storage_requirements:
    limits:
      storage: 10Gi              # Database storage
```

**Gateway:**
```yaml
automation_gateway_spec:
  replicas: 1                    # Number of gateway replicas
  route_tls_termination_mechanism: Edge  # TLS termination
```

After making changes, commit and push to Git. Argo CD will automatically sync.

### Repository Path Configuration

If you fork this repository or change its URL, update the repository reference in `bootstrap/root-app.yaml`:

```yaml
spec:
  source:
    repoURL: https://github.com/<your-org>/ocp-ansible-gitops.git  # Update this
    targetRevision: main
```

## Accessing AAP

Once deployed, access AAP components via their routes:

```bash
# List all routes
oc get routes -n ansible-automation-platform

# Get Automation Controller URL
oc get route <controller-route-name> -n ansible-automation-platform -o jsonpath='{.spec.host}'

# Get admin password
oc get secret <platform-name>-admin-password -n ansible-automation-platform -o jsonpath='{.data.password}' | base64 -d
```

Default credentials:
- **Username**: `admin` (as configured in platform.yaml)
- **Password**: Retrieved from secret (see command above)

## Troubleshooting

### Application Not Syncing

Check Argo CD Application status:

```bash
oc describe application aap-bootstrap -n openshift-gitops
oc describe application aap-instance -n openshift-gitops
```

### Operator Installation Failed

Check subscription and CSV status:

```bash
oc get subscription -n ansible-automation-platform
oc get csv -n ansible-automation-platform
oc describe csv -n ansible-automation-platform
```

### Platform Instance Not Creating

Check the AnsibleAutomationPlatform resource:

```bash
oc get ansibleautomationplatform -n ansible-automation-platform
oc describe ansibleautomationplatform <platform-name> -n ansible-automation-platform
```

Check operator logs:

```bash
oc logs -n ansible-automation-platform -l control-plane=controller-manager --tail=100
```

### RBAC Permission Issues

Verify the RoleBinding exists:

```bash
oc get rolebinding argocd-admin-binding -n ansible-automation-platform
```

If missing, reapply:

```bash
oc apply -k components/aap-operator/
```

## Deployment Sequence

The deployment follows this sequence:

1. **Bootstrap Applied** � Argo CD creates two Application resources
2. **Operator Deployed** � AAP Operator installed via `aap-bootstrap` app
3. **Operator Ready** � CRDs registered, operator pod running
4. **Instance Deployed** � AAP Platform CR created via `aap-instance` app
5. **Reconciliation** � Operator creates Gateway, Controller, Hub, Database
6. **Platform Ready** � All components running, routes accessible

## Maintenance

### Updating AAP Configuration

1. Edit files in `components/aap-instance/`
2. Commit and push changes
3. Argo CD automatically syncs (or manually sync via UI)

### Upgrading AAP Version

Update the operator channel in `components/aap-operator/kustomization.yaml`:

```yaml
resources:
  - https://github.com/redhat-cop/gitops-catalog/ansible-automation-platform/operator/overlays/stable-2.7  # Update version
```

### Backing Up Configuration

All configuration is stored in Git. To back up runtime data:

```bash
# Backup the platform CR
oc get ansibleautomationplatform -n ansible-automation-platform -o yaml > backup-platform.yaml

# Backup secrets
oc get secrets -n ansible-automation-platform -o yaml > backup-secrets.yaml
```

## Security Considerations

- **Secrets Management**: Admin passwords are auto-generated and stored in OpenShift secrets
- **TLS**: All routes use Edge termination by default
- **RBAC**: Minimal required permissions granted to Argo CD service account
- **Network Policies**: Consider implementing network policies for production environments
- **Container Security**: Images pulled from authenticated Red Hat registries

## References

- [Ansible Automation Platform Documentation](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/)
- [OpenShift GitOps Documentation](https://docs.openshift.com/container-platform/latest/cicd/gitops/understanding-openshift-gitops.html)
- [Red Hat CoP GitOps Catalog](https://github.com/redhat-cop/gitops-catalog)
- [Argo CD Documentation](https://argo-cd.readthedocs.io/)

## License

This repository contains configuration files for deploying Red Hat Ansible Automation Platform. Usage requires valid Red Hat subscriptions and adherence to Red Hat license terms.

## Contributing

To contribute improvements:

1. Fork the repository
2. Create a feature branch
3. Make changes and test in a development cluster
4. Submit a pull request with detailed description

## Support

For issues related to:
- **This repository**: Open an issue in this repo
- **AAP Platform**: Contact Red Hat Support
- **OpenShift GitOps**: Contact Red Hat Support
- **Argo CD**: Refer to upstream Argo CD documentation
