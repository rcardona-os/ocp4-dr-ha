- install odf operator (aws-base)
- create odf cluster (aws-base)
- install acm operator (aws-base)
- create hub cluster instance (aws-base)
- check that local cñuster is running
- install odf mch operator

- create dr cluster on aws-base

```bash
$ cat << EOF | oc apply -f -
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRCluster
metadata:
  name: aws-base
spec:
  region: us-west-1
  s3ProfileName: ""
  clusterFence: "Unfenced"
  cidrs:
    - 0.0.0.0/0
EOF
```

```bash
$ cat << EOF | oc apply -f -
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRCluster
metadata:
  name: azure-base
spec:
  region: westeurope
  s3ProfileName: ""
  clusterFence: "Unfenced"
  cidrs:
    - 0.0.0.0/0
EOF
```


===

- policy
 
```bash
$ cat << EOF | oc apply -f -
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRPolicy
metadata:
  name: app-ha-drpolicy
spec:
  drClusters:
    - aws-base
    - azure-base
  schedulingInterval: 1m
EOF
```



```bash
$ cat << EOF | oc --kubeconfig=inst/auth/kubeconfig create -f -
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRPlacementControl
metadata:
  name: image-uploader-drpc
  namespace: ha-demo
spec:
  drPolicyRef:
    name: app-ha-drpolicy
  preferredCluster: aws1
  placementRef:
    kind: Placement
    name: image-uploader-placement
    apiVersion: cluster.open-cluster-management.io/v1beta1
  pvcSelector:
    matchLabels:
      app: image-uploader
EOF
```


===
```bash
$ cat << EOF | oc apply -f -
apiVersion: multicluster.odf.openshift.io/v1alpha1
kind: MirrorPeer
metadata:
  name: odf-mirrorpeer
spec:
  items:
  - clusterName: aws-base
    storageClusterRef:
      name: ocs-storagecluster
      namespace: openshift-storage
  - clusterName: azure-base
    storageClusterRef:
      name: ocs-storagecluster
      namespace: openshift-storage
  schedulingInterval: 1m
  type: async
EOF
```
===
fresh

- on hub

```bash
$ cat << EOF | oc apply -f -
# volsync-addon.yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: volsync
  namespace: aws-base
spec:
  installNamespace: openshift-operators
---
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: volsync
  namespace: azure-base
spec:
  installNamespace: openshift-operators
EOF
```

- on azure

```bash
$ cat << EOF | oc apply -f -
# azure-volsync-addon.yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: volsync
  namespace: azure-base
spec:
  installNamespace: openshift-operators
EOF
```

===
following

# OpenShift Data Foundation Disaster Recovery Setup

# 🟢 Prerequisites:
# - ACM (Advanced Cluster Management) installed on the hub cluster (gcp-base)
# - ODF installed and default StorageClass configured on both aws-base (primary) and azure-base (DR)
# - VolSync available on both aws-base and azure-base

# -----------------------------------------------------------------------------
# 1️⃣  Create MirrorPeer (On the hub cluster: gcp-base)
cat <<EOF | oc apply -f -
apiVersion: multicluster.odf.openshift.io/v1alpha1
kind: MirrorPeer
metadata:
  name: odf-mirrorpeer
spec:
  items:
  - clusterName: aws-base
    storageClusterRef:
      name: ocs-storagecluster
      namespace: openshift-storage
  - clusterName: azure-base
    storageClusterRef:
      name: ocs-storagecluster
      namespace: openshift-storage
  schedulingInterval: 5m
  type: async
EOF

# Check status:
oc get mirrorpeer odf-mirrorpeer -o jsonpath='{.status.phase}{"\n"}'

# -----------------------------------------------------------------------------
# 2️⃣  Create DRCluster objects (On the hub cluster: gcp-base)

## aws-base
cat <<EOF | oc apply -f -
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRCluster
metadata:
  name: aws-base
spec:
  region: us-west-1
  s3ProfileName: s3profile-aws-base-ocs-storagecluster
  clusterFence: Unfenced
  cidrs:
    - 0.0.0.0/0
EOF

## azure-base
cat <<EOF | oc apply -f -
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRCluster
metadata:
  name: azure-base
spec:
  region: eu-west-1
  s3ProfileName: s3profile-azure-base-ocs-storagecluster
  clusterFence: Unfenced
  cidrs:
    - 0.0.0.0/0
EOF

# Validate status:
oc get drcluster aws-base -o json | jq -r '.status.conditions[] | select(.type=="Validated")'
oc get drcluster azure-base -o json | jq -r '.status.conditions[] | select(.type=="Validated")'

# -----------------------------------------------------------------------------
# 3️⃣  Create DRPolicy (On the hub cluster: gcp-base)
cat <<EOF | oc apply -f -
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRPolicy
metadata:
  name: app-ha-drpolicy
spec:
  drClusters:
    - aws-base
    - azure-base
  schedulingInterval: 5m
  replicationClassSelector:
    matchLabels:
      replication-class: async
EOF

# Check DRPolicy validation:
oc get drpolicy app-ha-drpolicy -o yaml | grep -A6 "type: Validated"

# -----------------------------------------------------------------------------
# ✅ At this point, DR framework is ready.
# Next steps are app deployment with Placement, DRPC, and VRG.

