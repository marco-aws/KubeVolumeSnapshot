
### Configure CSI driver IAM pre-req: 

https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html


### Install CSI DRIVER: 


```
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/alpha/?ref=master"
```

### create a random PVC and PV

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/examples/kubernetes/dynamic-provisioning/specs/storageclass.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/examples/kubernetes/dynamic-provisioning/specs/claim.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/examples/kubernetes/dynamic-provisioning/specs/pod.yaml
```

lazy version: 

```
helm install stable/prometheus --generate-name
```

### Install Snapshot Beta CRDs:

apply CRD: https://github.com/kubernetes-csi/external-snapshotter/tree/master/config/crd

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
```

### Install Snapshot Controller:

deploy controller: https://github.com/kubernetes-csi/external-snapshotter/tree/master/deploy/kubernetes/snapshot-controller

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```

(this goes under default namespace, edit accordingly)


### Test: 

```
kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
pvc-9a92b01e-3569-4b91-83a9-11df78c81d08   4Gi        RWO            Delete           Bound    default/ebs-claim   ebs-sc                  3m41s
kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ebs-claim   Bound    pvc-9a92b01e-3569-4b91-83a9-11df78c81d08   4Gi        RWO            ebs-sc         4m10s
kubectl exec -it app cat /data/out.txt | tail -n 3
Tue Jul 28 19:16:03 UTC 2020
Tue Jul 28 19:16:08 UTC 2020
Tue Jul 28 19:16:13 UTC 2020
```


### Create class:  

```
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotClass
metadata:
  name: test-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete
```

### Create a snapshot:

```
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshot
metadata:
  name: test-snapshot
spec:
  volumeSnapshotClassName: test-snapclass
  source:
    persistentVolumeClaimName: ebs-claim
```


verify the snapshot creation: 

```
kubectl get volumesnapshot
NAME            READYTOUSE   SOURCEPVC   SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS    SNAPSHOTCONTENT                                    CREATIONTIME   AGE
test-snapshot   false        ebs-claim                           4Gi           test-snapclass   snapcontent-8d8d1769-c41a-4cfc-8e4d-286564c868cc   40s            68s
kubectl get volumesnapshotcontent
NAME                                               READYTOUSE   RESTORESIZE   DELETIONPOLICY   DRIVER            VOLUMESNAPSHOTCLASS   VOLUMESNAPSHOT   AGE
snapcontent-8d8d1769-c41a-4cfc-8e4d-286564c868cc   true         4294967296    Delete           ebs.csi.aws.com   test-snapclass        test-snapshot    115s
```

### create a volume from a Snapshot
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-restore
  namespace: default
spec:
  storageClassName: ebs-sc
  dataSource:
    name: test-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```



### Deploy something that uses this PVC:


```
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: pvc-restore
      persistentVolumeClaim:
        claimName: pvc-restore
  containers:
    - name: task-pv-container
      image: centos
      command: ["/bin/sh"]
      args: ["-c", "sleep 100000"]
      volumeMounts:
        - mountPath: "/data"
          name: pvc-restore
```


### verify that the Volume contains the data: 


```
kubectl exec -it task-pv-pod -- cat /data/out.txt | tail -n 3
Tue Jul 28 19:17:48 UTC 2020
Tue Jul 28 19:17:53 UTC 2020
```


PVs:

```
kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
pvc-938f4c7f-9c04-489e-a3d1-d541ba46c54c   20Gi       RWO            Delete           Bound    default/pvc-restore   ebs-sc                  11m
pvc-9a92b01e-3569-4b91-83a9-11df78c81d08   4Gi        RWO            Delete           Bound    default/ebs-claim     ebs-sc                  22m
```
