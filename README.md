# ArgoCD Cluster Auto-Attach via Add-on Management System

Automatically register VKS workload clusters as ArgoCD target clusters using
the vSphere Supervisor Add-on Management System's `AddonConfigDefinition` (ACD)
process.

## Architecture

```mermaid
graph TD
    subgraph Supervisor
        ACD["AddonConfigDefinition<br/>(argocd-cluster-attach)"]
        AI["AddonInstall<br/>(selects VKS clusters)"]
        CA["Addon<br/>(per-cluster)"]
        AC["AddonConfig<br/>(per-cluster)"]
        KC["Secret: &lt;cluster&gt;-kubeconfig"]
        CS["ArgoCD Cluster Secret<br/>(in cluster namespace)"]
        SL["argocd-secret-labeler<br/>(adds argocd label)"]
        ARGO["ArgoCD"]

        ACD --> AC
        AI --> CA
        CA -- creates --> AC
        KC -- ".Dependencies.clusterKubeconfig" --> AC
        AC -- creates --> CS
        CS --> SL
        SL -- "labels secret" --> ARGO
    end

    ARGO -- "client certificates<br/>from kubeconfig" --> VKS["VKS Workload Cluster<br/>(target)"]
```

## How It Works

1. **AddonConfigDefinition** reads the VKS cluster's kubeconfig Secret from the
   Supervisor via `templateInputResources`, extracts the API server URL and CA
   certificate using `parseClusterInfo` / `getClusterServer`, and creates an
   ArgoCD-compatible cluster Secret in the cluster namespace.

2. **argocd-secret-labeler** (Deployment) polls for Secrets containing the
   `managed-by: argocd-cluster-attach` marker and adds the
   `argocd.argoproj.io/secret-type: cluster` label so ArgoCD discovers them.
   This is needed because `supervisorNamespaceOutput` does not support setting
   labels on the output resource.

## Files

| File | Description |
|------|-------------|
| `Addon-AddonRelease-AddonConfigDefinition.yaml` | Addon and AddonRelease metadata + ACD — extracts kubeconfig and creates ArgoCD cluster secret |
| `AddonInstall-AddonConfig-Cluster.yaml` | AddonInstall + example AddonConfig + sample Cluster def. |
| `argocd-secret-labeler.yaml` | Deployment that auto-labels secrets for ArgoCD discovery |

## Prerequisites

1. ArgoCD installed on the Supervisor (via ArgoCD Supervisor Service Operator)
2. VKS clusters provisioned with kubeconfig Secrets on the Supervisor

## Installation

### Step 1: Apply the ACD and addon metadata

```bash
kubectl apply -f Addon-AddonRelease-AddonConfigDefinition.yaml
```

### Step 2: Deploy the secret labeler

The labeler runs as a lightweight Deployment and automatically labels secrets
created by the ACD, so ArgoCD discovers them.

```bash
kubectl apply -f argocd-secret-labeler.yaml
```

### Step 3: Install the addon on target clusters

Edit `AddonInstall-AddonConfig-Cluster.yaml` to select the desired namespace and clusters, then:

```bash
kubectl apply -f AddonInstall-AddonConfig-Cluster.yaml
```

## Verification

```bash
# Check secret exists and has the label
kubectl get secret -n demo1 -l argocd.argoproj.io/secret-type=cluster

# Check secret content
kubectl get secret -n demo1 argocd-cluster-workload-vsphere-vks1 \
  -o jsonpath='{.data.server}' | base64 -d

# Check ArgoCD sees the cluster
argocd cluster list
```

### Known ACD limitations

- `supervisorNamespaceOutput` only supports `apiVersion`, `kind`, `name` — no
  `labels`, `metadata`, or `annotations`
- `targetClusterOutput` only supports `v1/Secret` resources
- `constraints` on `templateInputResources` only supports `optional` operator
- Template body cannot set `metadata.labels` — labels are silently dropped
- `getClusterServer` requires a `*api.Cluster` (from `parseClusterInfo`), not a string
- `CertificateAuthorityData` from `parseClusterInfo` is `[]byte` — use
  `printf "%s"` then `encodebase64` to get a base64 string
