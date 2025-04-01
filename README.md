# OpenShift Data Foundation (ODF) Disaster Recovery (DR) Setup Using ACM & MCO

This guide outlines step-by-step instructions for setting up ODF DR across two OpenShift clusters (`aws-base` and `azure-base`) using ACM, ODF Multicluster Orchestrator, and Ramen DR components.

---

## Step 1: Install Required Operators on Both Clusters

### On `aws-base` (Hub Cluster)
```sh
# Install OpenShift Data Foundation (ODF)
# Install Advanced Cluster Management (ACM)
# Install ODF Multicluster Orchestrator (MCO)
```

### On `azure-base` (Spoke Cluster)
```sh
# Install OpenShift Data Foundation (ODF)
# Install ACM (only the operator, do not configure MCH)
```

---

## Step 2: Configure MultiClusterHub (MCH) on aws-base
```sh
# Create the MultiClusterHub resource on aws-base to initialize ACM hub functionalities
```

---

## Step 3: Import azure-base as a Managed Cluster

### On `aws-base`
```sh
# Retrieve import.yaml bundle from ACM UI or CLI:
oc get managedcluster azure-base -o jsonpath='{.status.consoleURL}'

# Apply import manifest on azure-base:
oc apply -f import.yaml
```

---

## Step 4: Verify Managed Cluster States
```sh
oc get managedclusters
oc get managedclusteraddons -n azure-base
```
Ensure both `aws-base` and `azure-base` show `Available=True`.

---

## Step 5: Create MirrorPeer

### On `aws-base`
```yaml
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
  schedulingInterval: 2m
  type: async
```
Apply it:
```sh
oc apply -f mirrorpeer.yaml
```
Check status:
```sh
oc get mirrorpeer odf-mirrorpeer -o json | jq -r '.status.phase'
```
Status should change from `ExchangingSecret` to `ExchangedSecret`.

---

## Step 6: Deploy DRClusters

### On `aws-base`
```yaml
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
```

```yaml
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
```

Check validation:
```sh
oc get drcluster aws-base azure-base -o yaml | grep -A5 'type: Validated'
```

---

## Step 7: Ensure VolSync AddOn is Deployed

If not automatically installed:
```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: volsync
  namespace: aws-base
spec:
  installNamespace: openshift-operators
```
Apply it and check status:
```sh
oc apply -f volsync-addon.yaml
oc get managedclusteraddons -n aws-base volsync -o yaml
```
Look for status `Available=True`

---

## Step 8: Create DRPolicy
```yaml
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRPolicy
metadata:
  name: app-ha-drpolicy
spec:
  drClusters:
    - aws-base
    - azure-base
  replicationClassSelector:
    matchLabels:
      replication-class: async
  schedulingInterval: 2m
```
Apply:
```sh
oc apply -f drpolicy.yaml
```

---

## Step 9: Deploy Application & DR Resources

Apply the following in order:
1. `0-namespace.yaml`
2. `1-pvc.yaml`
3. `2-deployment.yaml`
4. `3-service.yaml`
5. `4-placement.yaml` (ensure it includes label `ramendr.openshift.io/ramen-enabled=true`)
6. `5-drplacementcontrol.yaml`

Annotate Placement:
```sh
oc -n ha-demo annotate placement image-uploader-placement cluster.open-cluster-management.io/disable-decision="true"
```

Verify:
```sh
oc get vrg -n ha-demo
oc get drpc -n ha-demo
```

---

## Step 10: Troubleshooting Tips

- Check if VRG is created: `oc get vrg -n ha-demo`
- Check DRPlacementControl status: `oc get drpc -n ha-demo -o yaml`
- Confirm MirrorPeer status: `oc get mirrorpeer odf-mirrorpeer -o json | jq -r '.status.phase'`
- If DRCluster is stuck, patch metadata or re-apply the manifest.

---

## Final Notes
- Keep all components healthy and double check namespaces, CRDs, and VolSync availability.
- Use `oc describe` and `oc logs` for deeper troubleshooting when resources are not behaving as expected.

---

> This guide helps ensure a consistent and repeatable DR configuration process for ODF + ACM based setups.
