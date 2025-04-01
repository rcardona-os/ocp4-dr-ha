- install odf operator (aws-base)
- create odf cluster (aws-base)
- install acm operator (aws-base)
- create hub cluster instance (aws-base)
- check that local cñuster is running
- install odf mch operator



```bash
$ cat << EOF | oc --kubeconfig=inst/auth/kubeconfig create -f -
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRPolicy
metadata:
  name: app-ha-drpolicy
spec:
  drClusters:
    - aws1
    - az2
  schedulingInterval: 1m
EOF
```

```bash
$ cat << EOF | oc --kubeconfig=inst/auth/kubeconfig create -f -
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRCluster
metadata:
  name: aws1
spec:
  region: eu-west-1
  s3ProfileName: ""
  clusterFence: "Unfenced"
  cidrs:
    - 0.0.0.0/0
EOF
```

```bash
$ cat << EOF | oc --kubeconfig=inst/auth/kubeconfig create -f -
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRCluster
metadata:
  name: az2
spec:
  region: westeurope
  s3ProfileName: ""
  clusterFence: "Unfenced"
  cidrs:
    - 0.0.0.0/0
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