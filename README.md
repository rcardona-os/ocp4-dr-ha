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
  region: us-west-1
  s3ProfileName: ""
  clusterFence: False
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
spec
  region: westeurope
  s3ProfileName: ""
  clusterFence: False
  cidrs:
    - 0.0.0.0/0
EOF
```