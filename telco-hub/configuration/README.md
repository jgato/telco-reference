
# Automated installation

The Telco Hub reference design provides a set of validated Custom Resources (CRs) in the `reference-crs/` directory. Rather than modifying those resources directly, the recommended approach is to use the Kustomize overlay layer in `example-overlays-config/` to apply your environment-specific customizations on top of them. An ArgoCD Application then deploys the full Telco Hub and continuously reconciles it via GitOps.

Refer to the [Telco Hub RDS documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/scalability_and_performance/telco-hub-ref-design-specs) for details on which components are optional or mandatory.

---

## Prerequisites

- An OpenShift Container Platform (OCP) cluster is up and running.
- OpenShift GitOps (ArgoCD) is installed on the cluster (see [Init phase](#init-phase-install-argocd-or-openshift-gitops) below if needed).
- If using ODF in "internal" mode, nodes with available storage must be labeled:
  ```bash
  oc label node master-0 cluster.ocs.openshift.io/openshift-storage=
  oc label node master-1 cluster.ocs.openshift.io/openshift-storage=
  oc label node master-2 cluster.ocs.openshift.io/openshift-storage=
  ```
- Configured and existing OpenShift CatalogSources for `redhat-operators-disconnected` and `certified-operators-disconnected` (disconnected environments).

---

## Init phase (install ArgoCD or OpenShift GitOps)

This phase is optional if you already have ArgoCD or OpenShift GitOps running on your cluster.

ArgoCD is one of the main components of the Telco Hub: it manages deployment and configuration of the infrastructure using a GitOps methodology. At the same time, we use ArgoCD itself to deploy the Telco Hub (the recommended procedure). Therefore, to have a Telco Hub with ArgoCD, you first need ArgoCD running. This is the init phase.

To install ArgoCD using the existing `reference-crs` for GitOps:

```bash
oc apply -f reference-crs/required/gitops/clusterrole.yaml \
  -f reference-crs/required/gitops/clusterrolebinding.yaml \
  -f reference-crs/required/gitops/gitopsNS.yaml \
  -f reference-crs/required/gitops/gitopsOperatorGroup.yaml \
  -f reference-crs/required/gitops/gitopsSubscription.yaml
```

Wait for the operator to be installed:

```bash
oc -n openshift-gitops-operator get subscriptions.operators.coreos.com \
  openshift-gitops-operator -o jsonpath='{.status.state}'
# Expected: AtLatestKnown

oc -n openshift-gitops get pod
# All pods should be Running
```

> **Note:** The reference configuration includes a ClusterRole for ArgoCD which grants the necessary permissions for installing the remainder of the reference. This updates the currently running ArgoCD application to allow it to complete the full synchronization.

---

## Step 1 -- Fork and set up your overlay

Fork this repository and clone your fork. All your customizations will live in the overlay directory, keeping the upstream `reference-crs/` untouched.

Rename the example overlay to your own:

```bash
cd telco-hub/configuration/
mv example-overlays-config/ my-hub-overlay/
```

Update the root `kustomization.yaml` references accordingly:

```bash
sed -i 's/example-overlays-config/my-hub-overlay/g' kustomization.yaml
```

The root `kustomization.yaml` controls which components are deployed. Comment or uncomment entries depending on your environment:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # (Optional) If you use LocalStorage operator, edit and configure the patch
  - my-hub-overlay/lso/

  # (Optional) If you use ODF, edit and configure storage settings
  - my-hub-overlay/odf/

  # Required overlays
  - my-hub-overlay/gitops/
  - my-hub-overlay/acm/
  - my-hub-overlay/registry/

  # Mandatory resources not managed by any overlay
  - reference-crs/required/talm/

  # (Optional) Include ArgoCD configuration for GitOps ZTP management
  # of cluster installation and configuration
  # - reference-crs/required/gitops/ztp-installation
```

---

## Step 2 -- Configure your environment patches

Each component directory contains patch files that must be updated with your environment-specific values. Work through each component you have enabled.

### (Optional) LocalStorage Operator

Edit `my-hub-overlay/lso/local-storage-disks-patch.yaml` to specify the physical disk devices on your nodes:

```yaml
- op: replace
  path: /spec/storageClassDevices/0/devicePaths
  value:
    - /dev/nvme1n1
```

### (Optional) ODF Storage

Edit `my-hub-overlay/odf/options-storage-cluster.yaml` to set storage capacity and storage class:

```yaml
- op: replace
  path: /spec/storageDeviceSets/0/dataPVCTemplate/spec/resources/requests/storage
  value: "400Gi"

- op: replace
  path: /spec/storageDeviceSets/0/dataPVCTemplate/spec/storageClassName
  value: "local-sc"
```

### Registry (disconnected environments)

Edit the patch files in `my-hub-overlay/registry/` to configure your mirror registry:

- `catalog-source-image-patch.yaml` -- Set the CatalogSource image URL pointing to your mirror:
  ```yaml
  - op: replace
    path: /spec/image
    value: <registry.example.com:8443>/openshift-marketplace/redhat-operators-disconnected:v4.x
  ```

- `registry-ca-patch.yaml` -- Add the CA certificate for your mirror registry:
  ```yaml
  - op: replace
    path: "/data"
    value:
      registry.example.com..8443: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
  ```

- `idms-operator-mirrors-patch.yaml`, `idms-release-mirrors-patch.yaml`, `itms-generic-mirrors-patch.yaml`, `itms-release-mirrors-patch.yaml` -- Update the mirror entries to point to your registry.

### MultiClusterObservability Storage

Edit `my-hub-overlay/acm/storage-mco-patch.yaml` to select a filesystem StorageClass:

```yaml
- op: replace
  path: /spec/storageConfig/storageClass
  value: "ocs-storagecluster-cephfs"
```

### AgentServiceConfig

Edit `my-hub-overlay/acm/options-agentserviceconfig-patch.yaml` to configure storage classes and RHCOS image URLs. In disconnected environments, image URLs must point to your internal mirror registry:

```yaml
- op: replace
  path: /spec/databaseStorage/storageClassName
  value: "ocs-storagecluster-cephfs"

- op: replace
  path: /spec/filesystemStorage/storageClassName
  value: "ocs-storagecluster-cephfs"

- op: replace
  path: /spec/imageStorage/storageClassName
  value: "ocs-storagecluster-cephfs"

- op: replace
  path: "/spec/osImages"
  value:
    - cpuArchitecture: x86_64
      openshiftVersion: "4.x"
      rootFSUrl: https://mirror.example.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.x/latest/rhcos-live-rootfs.x86_64.img
      url: https://mirror.example.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.x/latest/rhcos-live-iso.x86_64.iso
      version: <rhcos-version>
```

In a connected environment, remove the `mirrorRegistryRef` from the spec to avoid routing spoke clusters through an internal registry unnecessarily. Uncomment the following in the patch file:

```yaml
- op: remove
  path: /spec/mirrorRegistryRef
```

### GitOps TLS certificates

If your git server uses a custom TLS certificate, edit `my-hub-overlay/gitops/argocd-tls-certs-cm-patch.yaml` to add the server certificate:

```yaml
- op: replace
  path: "/data"
  value:
    git.example.com: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
```

### Other overlays

Additional overlay directories are available for optional components:

- **cert-manager** (`my-hub-overlay/cert-manager/`) -- ACME issuer, ingress and API server certificates. See the [cert-manager overlay README](example-overlays-config/cert-manager/README.md) for details.
- **logging** (`my-hub-overlay/logging/`) -- ClusterLogForwarder configuration (e.g., Kafka endpoint). See the [logging overlay README](example-overlays-config/logging/README.md) for details.

To enable these, add them as resources in the root `kustomization.yaml`:

```yaml
  - my-hub-overlay/cert-manager/
  - my-hub-overlay/logging/
```

---

## Step 3 -- Configure the hub-config ArgoCD Application

The `hub-config` ArgoCD Application is the central piece that ties everything together. It must point to **your fork**.

Edit `my-hub-overlay/gitops/init-argocd-app.yaml`:

```yaml
- op: replace
  path: "/spec/source"
  value:
    repoURL: "https://github.com/<your-org>/telco-reference.git"
    path: "telco-hub/configuration"
    targetRevision: "main"
```

Adjust `repoURL` and `targetRevision` to match your fork and branch.

## Step 4 -- Validate your overlay with kustomize build

Before bootstrapping ArgoCD, do a dry run to ensure your overlay builds without errors:

```bash
kustomize build .
```

Review the rendered output and confirm all patches were applied correctly and no unexpected values remain from the example template.

If everything looks good, commit and push all your changes.

---

## Step 5 -- Bootstrap the hub-config ArgoCD Application

Once all patches are configured and pushed, bootstrap the `hub-config` Application:

```bash
kustomize build my-hub-overlay/gitops/ | oc apply -f -
```

From this point on, ArgoCD takes full ownership. It will continuously reconcile the cluster against your overlay, deploying the complete Telco Hub configuration. Any future changes should be made in git -- not directly on the cluster.

## Sync-Wave Ordering

All resources in the `reference-crs` directory are configured with ArgoCD sync-wave annotations to ensure deterministic and reliable rollout. The deployment follows this sequence:

1. **Registry Foundation** (sync-wave -50): Registry and catalog configurations
2. **Namespaces** (sync-wave -45): All namespace definitions
3. **Namespaced Resources** (sync-wave -40): RBAC, ConfigMaps, OperatorGroups, Subscriptions
4. **ArgoCD Resources** (sync-wave -35): AppProjects and Applications
5. **Independent Custom Resources** (sync-wave -30): Core operators, infrastructure, and storage deployment
6. **Policies and Validation** (sync-wave -25): ACM policies for configuration and infrastructure validation
7. **Storage-Dependent Services** (sync-wave -10): Services requiring validated storage
8. **ZTP Components** (sync-wave 100): Zero Touch Provisioning resources deployed last

This ordering ensures all dependencies are properly resolved and aims at preventing race conditions during deployment.

For detailed information about the sync-wave implementation, design principles, and complete file listings, see [SYNC-WAVES.md](SYNC-WAVES.md).

---

## Validating compliance with the reference design

Adding custom patches may cause your cluster to deviate from the Telco Hub Reference Design. To verify compliance at any time, use the `cluster-compare` plugin against the `reference-crs-kube-compare` directory:

```bash
kubectl cluster-compare -r https://github.com/openshift-kni/telco-reference//telco-hub/configuration/reference-crs-kube-compare
```

This tool compares the live cluster state against the reference and reports any deviations.
