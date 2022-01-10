# 1. Pre-requisites

* Kublr 1.21.2
* jq
* kubectl 1.21

# 2. Deploy cluster

Create clusters in Kublr UI or API.

Wait for all statuses become green, including 4 packages in the end of the status page - `rook-ceph-additional-configuration`, `rook-ceph`, `rook-ceph-cluster`, `submariner-k8s-broker`

# 3. Download clusters' kubeconfig files

Save the clusters' kubeconfig files into `config1aws` and `config2az`

# 4. Check that Rook/Ceph is up

Access Ceph UI - see more info on https://rook.io/docs/rook/v1.7/ceph-dashboard.html

```
echo
echo "URL: http://$(kubectl get -n kube-system svc kublr-ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")/ceph-dashboard/"
echo "Password: $(kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode)"
```

# 5. Connect clusters with submariner

## 5.1. Make sure that `subctl` is installed:

```
wget https://github.com/submariner-io/submariner-operator/releases/download/subctl-release-0.11/subctl-release-0.11-linux-amd64.tar.xz
tar xfv subctl-release-0.11-linux-amd64.tar.xz
mv subctl-release-0.11/subctl-release-0.11-linux-amd64 subctl
```

## 5.2. Prepare common submariner parameters:

```
export KUBECONFIG=config1aws

export BROKER_NS=submariner-k8s-broker
export SUBMARINER_NS=submariner-operator
export SUBMARINER_PSK=$(LC_CTYPE=C tr -dc 'a-zA-Z0-9' < /dev/urandom | fold -w 64 | head -n 1)

export SUBMARINER_BROKER_CA=$(kubectl -n "${BROKER_NS}" get secrets \
    -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='${BROKER_NS}-client')].data['ca\.crt']}")

export SUBMARINER_BROKER_TOKEN=$(kubectl -n "${BROKER_NS}" get secrets \
    -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='${BROKER_NS}-client')].data.token}" \
       | base64 --decode)

export SUBMARINER_BROKER_URL="$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}' | grep -E -o -i '([^/]+):[0-9]+')"

echo
echo "KUBECONFIG=${KUBECONFIG}" ; echo
echo "BROKER_NS=${BROKER_NS}" ; echo
echo "SUBMARINER_NS=${SUBMARINER_NS}" ; echo
echo "SUBMARINER_PSK=${SUBMARINER_PSK}" ; echo
echo "SUBMARINER_BROKER_CA=${SUBMARINER_BROKER_CA}" ; echo
echo "SUBMARINER_BROKER_TOKEN=${SUBMARINER_BROKER_TOKEN}" ; echo
echo "SUBMARINER_BROKER_URL=${SUBMARINER_BROKER_URL}" ; echo
```

## 5.3. Join AWS cluster

Deploy submariner operator to the AWS cluster

```
export KUBECONFIG=config1aws
export CLUSTER_ID=demo-hybrid-1-aws
export CLUSTER_CIDR=100.96.0.0/11
export SERVICE_CIDR=100.64.0.0/13

echo
echo "KUBECONFIG=${KUBECONFIG}" ; echo
echo "CLUSTER_ID=${CLUSTER_ID}" ; echo
echo "CLUSTER_CIDR=${CLUSTER_CIDR}" ; echo
echo "SERVICE_CIDR=${SERVICE_CIDR}" ; echo

helm upgrade -i submariner-operator https://submariner-io.github.io/submariner-charts/charts/submariner-operator-0.11.0.tgz \
        --create-namespace \
        --namespace "${SUBMARINER_NS}" \
        --set ipsec.psk="${SUBMARINER_PSK}" \
        --set broker.server="${SUBMARINER_BROKER_URL}" \
        --set broker.token="${SUBMARINER_BROKER_TOKEN}" \
        --set broker.namespace="${BROKER_NS}" \
        --set broker.ca="${SUBMARINER_BROKER_CA}" \
        --set submariner.cableDriver=libreswan \
        --set submariner.clusterId="${CLUSTER_ID}" \
        --set submariner.clusterCidr="${CLUSTER_CIDR}" \
        --set submariner.serviceCidr="${SERVICE_CIDR}" \
        --set submariner.globalCidr="${GLOBAL_CIDR}" \
        --set serviceAccounts.globalnet.create="" \
        --set submariner.natEnabled="true" \
        --set brokercrd.create=false
```

## 5.4. Join Azure cluster

Deploy submariner operator to the Azure cluster

```
export KUBECONFIG=config2az
export CLUSTER_ID=demo-hybrid-2-azure
export CLUSTER_CIDR=100.160.0.0/11
export SERVICE_CIDR=100.128.0.0/13

echo
echo "KUBECONFIG=${KUBECONFIG}" ; echo
echo "CLUSTER_ID=${CLUSTER_ID}" ; echo
echo "CLUSTER_CIDR=${CLUSTER_CIDR}" ; echo
echo "SERVICE_CIDR=${SERVICE_CIDR}" ; echo

helm upgrade -i submariner-operator https://submariner-io.github.io/submariner-charts/charts/submariner-operator-0.11.0.tgz \
        --create-namespace \
        --namespace "${SUBMARINER_NS}" \
        --set ipsec.psk="${SUBMARINER_PSK}" \
        --set broker.server="${SUBMARINER_BROKER_URL}" \
        --set broker.token="${SUBMARINER_BROKER_TOKEN}" \
        --set broker.namespace="${BROKER_NS}" \
        --set broker.ca="${SUBMARINER_BROKER_CA}" \
        --set submariner.cableDriver=libreswan \
        --set submariner.clusterId="${CLUSTER_ID}" \
        --set submariner.clusterCidr="${CLUSTER_CIDR}" \
        --set submariner.serviceCidr="${SERVICE_CIDR}" \
        --set submariner.globalCidr="${GLOBAL_CIDR}" \
        --set serviceAccounts.globalnet.create="" \
        --set submariner.natEnabled="true" \
        --set brokercrd.create=false
```

Wait for VPN connection to be set up

```
./subctl show all
```

## 5.5. Test inter-pod connectivity

Deploy test server pods

```
KUBECONFIGS="config1aws config2az"
for C in $KUBECONFIGS ; do
kubectl --kubeconfig=$C apply -f - <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: test-srv
  labels:
    app: test-srv
spec:
  selector:
    matchLabels:
      app: test-srv
  template:
    metadata:
      labels:
        app: test-srv
    spec:
      containers:
        - name: srv
          image: quay.io/gravitational/netbox:latest
      terminationGracePeriodSeconds: 1
      tolerations:
        - operator: Exists
EOF
done
```

Wait for all pods to start on all nodes

```
kubectl --kubeconfig=config1aws get pods -l app=test-srv ; kubectl --kubeconfig=config2az get pods -l app=test-srv
```

Test that inter-pod connections work

```
for C in $KUBECONFIGS ; do
  for P in $(kubectl --kubeconfig=$C get pods -l app=test-srv -o name); do
    for A in $(for C in $KUBECONFIGS ; do kubectl --kubeconfig=$C get pods -l app=test-srv -o jsonpath='{.items[*].status.podIP}' ; echo ; done); do
      echo -n "$P -> $A: "
      kubectl --kubeconfig=$C exec $P -- curl -q http://$A:5000 1>/dev/null 2>&1 && echo OK || echo NOK
    done
  done
done
```

# 6. Data mirroring and DR failover

## 6.1. Enable mirroring and connect the main block pool

Enable mirroring

```
kubectl --kubeconfig=config1aws -n rook-ceph patch cephblockpool ceph-blockpool --type merge -p '{"spec":{"mirroring":{"enabled":true,"mode":"image"}}}'
kubectl --kubeconfig=config2az  -n rook-ceph patch cephblockpool ceph-blockpool --type merge -p '{"spec":{"mirroring":{"enabled":true,"mode":"image"}}}'
```

Configure the pools as each others' mirrors

```
export KUBECONFIG=`pwd`/config1aws
S1="$(kubectl get cephblockpool.ceph.rook.io/ceph-blockpool -n rook-ceph -ojsonpath='{.status.info.rbdMirrorBootstrapPeerSecretName}')"
T1="$(kubectl get secret -n rook-ceph $S1 -o jsonpath='{.data.token}'| base64 -d)"
export KUBECONFIG=`pwd`/config2az
S2="$(kubectl get cephblockpool.ceph.rook.io/ceph-blockpool -n rook-ceph -ojsonpath='{.status.info.rbdMirrorBootstrapPeerSecretName}')"
T2="$(kubectl get secret -n rook-ceph $S2 -o jsonpath='{.data.token}'| base64 -d)"
echo S1=$S1
echo T1=$T1
echo S2=$S2
echo T2=$T2

kubectl --kubeconfig=config1aws -n rook-ceph create secret generic rbd-primary-site-secret --from-literal=token="$T2" --from-literal=pool=ceph-blockpool
kubectl --kubeconfig=config2az  -n rook-ceph create secret generic rbd-primary-site-secret --from-literal=token="$T1" --from-literal=pool=ceph-blockpool

KUBECONFIGS="config1aws config2az"
for C in $KUBECONFIGS ; do
kubectl --kubeconfig=$C -n rook-ceph patch cephblockpool ceph-blockpool --type merge -p '{"spec":{"mirroring":{"peers": {"secretNames": ["rbd-primary-site-secret"]}}}}'
kubectl --kubeconfig=$C apply -f - <<EOF
apiVersion: replication.storage.openshift.io/v1alpha1
kind: VolumeReplicationClass
metadata:
  name: rbd-volumereplicationclass
spec:
  provisioner: rook-ceph.rbd.csi.ceph.com
  parameters:
    mirroringMode: snapshot
    schedulingInterval: "1m"
    replication.storage.openshift.io/replication-secret-name: rook-csi-rbd-provisioner
    replication.storage.openshift.io/replication-secret-namespace: rook-ceph
EOF
done
```

Check mirroring status and wait for "heathy"

```
for C in $KUBECONFIGS ; do
kubectl --kubeconfig=$C get cephblockpools.ceph.rook.io ceph-blockpool -n rook-ceph -o jsonpath='{.status.mirroringStatus.summary}' ; echo
done
```

## 6.2. Deploy stateful application in AWS cluster

Deploy application using an RBD image PV

```
export KUBECONFIG=config1aws

kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-block
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blocktest
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blocktest
  template:
    metadata:
      labels:
        app: blocktest
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - name: main
        image: ubuntu
        command: ["/bin/sh", -c, "while true; do touch /mnt/fs/file; date; cat /mnt/fs/file; sleep 2; done"]
        volumeMounts:
        - name: fs
          mountPath: /mnt/fs
      volumes:
      - name: fs
        persistentVolumeClaim:
          claimName: block-pvc
          readOnly: false
EOF
```

Enable mirroring for the volume

```
kubectl apply -f - <<EOF
apiVersion: replication.storage.openshift.io/v1alpha1
kind: VolumeReplication
metadata:
  name: block-pvc-volumereplication
spec:
  volumeReplicationClass: rbd-volumereplicationclass
  replicationState: primary
  dataSource:
    apiGroup: ""
    kind: PersistentVolumeClaim
    name: block-pvc
EOF
```

Check mirroring status

```
kubectl get volumereplication block-pvc-volumereplication -oyaml
```

Update data on the volume and check that the application sees it

```
kubectl exec -it         deployment/blocktest -- sh -c 'echo $@ > /mnt/fs/file ; sync' sh Hello world 1
kubectl logs --tail=5 -f deployment/blocktest
```

## 6.3. Prepare application recovery in Azure cluster

Copy application PV and PVC definitions from AWS to Azure cluster

```
kubectl --kubeconfig=config1aws get pv $(kubectl --kubeconfig=config1aws get pvc block-pvc -o jsonpath='{.spec.volumeName}') -o json |
  jq 'del(.spec.claimRef,.metadata.resourceVersion,.metadata.uid,.metadata.creationTimestamp,.status)' |
  kubectl --kubeconfig=config2az apply -f -

kubectl --kubeconfig=config1aws get pvc block-pvc -o json |
  kubectl --kubeconfig=config2az apply -f -

kubectl --kubeconfig=config1aws get VolumeReplication block-pvc-volumereplication -o json |
  jq 'del(.metadata.resourceVersion,.metadata.uid,.metadata.creationTimestamp,.status)|.spec.replicationState="secondary"' |
  kubectl --kubeconfig=config2az apply -f -

kubectl --kubeconfig=config1aws get deployment blocktest -o json |
  jq 'del(.metadata.resourceVersion,.metadata.uid,.metadata.creationTimestamp,.status)|.spec.replicas=0' |
  kubectl --kubeconfig=config2az apply -f -
```

Check mirroring status

```
export KUBECONFIG=config2az
kubectl get volumereplication block-pvc-volumereplication -oyaml
```

## 6.4. Switch to the backup in Azure

Demote in AWS

```
export KUBECONFIG=config1aws
kubectl scale deployment blocktest --replicas=0
kubectl patch VolumeReplication block-pvc-volumereplication --type merge -p '{"spec":{"replicationState":"secondary"}}'
```

Check AWS Ceph UI or use kubectl CLI and wait until the image is demoted there.

Promote in Azure

```
export KUBECONFIG=config2az
kubectl patch VolumeReplication block-pvc-volumereplication --type merge -p '{"spec":{"replicationState":"primary"}}'
kubectl scale deployment blocktest --replicas=1
```


Check that the application is started, sees replicated data, and update data on the volume again

```
kubectl logs --tail=5 -f deployment/blocktest
kubectl exec -it         deployment/blocktest -- sh -c 'echo $@ > /mnt/fs/file ; sync' sh Hello world 2
```

## 6.5. Switch back to AWS

Scale down app and demote the image in Azure:

```
export KUBECONFIG=config2az
kubectl scale deployment blocktest --replicas=0
kubectl patch VolumeReplication block-pvc-volumereplication --type merge -p '{"spec":{"replicationState":"secondary"}}'
```

Check Azure Ceph UI or use kubectl CLI and wait until the image is demoted there.

Promote in AWS

```
export KUBECONFIG=config1aws
kubectl patch VolumeReplication block-pvc-volumereplication --type merge -p '{"spec":{"replicationState":"primary"}}'
kubectl scale deployment blocktest --replicas=1

kubectl logs --tail=5 -f deployment/blocktest
```

# 7. Consume CephFS and Block images

The following tests can be run in any of the clusters

## 7.1. Consume CephFS

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-filesystem
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sharedfstest
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sharedfstest
  template:
    metadata:
      labels:
        app: sharedfstest
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - name: main
        image: ubuntu
        command: ["/bin/sh", -c, "sleep 72000"]
        volumeMounts:
        - name: fs
          mountPath: /mnt/fs
      volumes:
      - name: fs
        persistentVolumeClaim:
          claimName: cephfs-pvc
          readOnly: false
EOF
```

Create some content on the FS volume:

```
kubectl exec -it deployment/sharedfstest -- sh -c 'echo $@ > /mnt/fs/file ; sync' sh Hello world "$(date)"
```

Check the content on the FS volume:

```
kubectl exec -it deployment/sharedfstest -- sh -c 'cat /mnt/fs/file'
```

## 7.2. Consume Block image

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-block
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: imagetest
spec:
  replicas: 1
  selector:
    matchLabels:
      app: imagetest
  template:
    metadata:
      labels:
        app: imagetest
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - name: main
        image: ubuntu
        command: ["/bin/sh", -c, "sleep 72000"]
        volumeMounts:
        - name: fs
          mountPath: /mnt/fs
      volumes:
      - name: fs
        persistentVolumeClaim:
          claimName: image-pvc
          readOnly: false
EOF
```

Create some content on the block volume:

```
kubectl exec -it deployment/imagetest -- sh -c 'echo $@ > /mnt/fs/file ; sync' sh Hello world "$(date)"
```

Check the content on the block volume:

```
kubectl exec -it deployment/imagetest -- sh -c 'cat /mnt/fs/file'
```

# 8. Work with snapshots

## 8.1. Create CephFS snapshot

```
kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-cephfsplugin-snapclass
driver: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
deletionPolicy: Delete
EOF

kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: cephfs-pvc-snapshot1
spec:
  volumeSnapshotClassName: csi-cephfsplugin-snapclass
  source:
    persistentVolumeClaimName: cephfs-pvc
EOF
```

Check snapshot

```
kubectl get VolumeSnapshot
```

Update the content on the FS volume:

```
kubectl exec -it deployment/sharedfstest -- sh -c 'echo $@ > /mnt/fs/file ; sync' sh Hello world "$(date)" '(after snapshot)'
kubectl exec -it deployment/sharedfstest -- sh -c 'cat /mnt/fs/file'
```

## 8.2. Create RBD snapshot

```
kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-rbdplugin-snapclass
driver: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
deletionPolicy: Delete
EOF

kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: image-pvc-snapshot1
spec:
  volumeSnapshotClassName: csi-rbdplugin-snapclass
  source:
    persistentVolumeClaimName: image-pvc
EOF
```

Check snapshot

```
kubectl get VolumeSnapshot
```

Update the content on the block volume:

```
kubectl exec -it deployment/imagetest -- sh -c 'echo $@ > /mnt/fs/file ; sync' sh Hello world "$(date)" '(after snapshot)'
kubectl exec -it deployment/imagetest -- sh -c 'cat /mnt/fs/file'
```

## 8.3. Use CephFS snapshot to recover volume

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc-copy1
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-filesystem
  dataSource:
    name: cephfs-pvc-snapshot1
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sharedfstest1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sharedfstest1
  template:
    metadata:
      labels:
        app: sharedfstest1
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - name: main
        image: ubuntu
        command: ["/bin/sh", -c, "sleep 72000"]
        volumeMounts:
        - name: fs
          mountPath: /mnt/fs
      volumes:
      - name: fs
        persistentVolumeClaim:
          claimName: cephfs-pvc-copy1
          readOnly: false
EOF
```

Check the content on the FS volume created from snapshot as well as the original volume:

```
kubectl exec -it deployment/sharedfstest1 -- sh -c 'cat /mnt/fs/file'
kubectl exec -it deployment/sharedfstest  -- sh -c 'cat /mnt/fs/file'
```

## 8.4. Use RBD snapshot to recover volume

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-pvc-copy1
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-block
  dataSource:
    name: image-pvc-snapshot1
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: imagetest1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: imagetest1
  template:
    metadata:
      labels:
        app: imagetest1
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - name: main
        image: ubuntu
        command: ["/bin/sh", -c, "sleep 72000"]
        volumeMounts:
        - name: fs
          mountPath: /mnt/fs
      volumes:
      - name: fs
        persistentVolumeClaim:
          claimName: image-pvc-copy1
          readOnly: false
EOF
```

Check the content on the block volume as well as the original volume:

```
kubectl exec -it deployment/imagetest1 -- sh -c 'cat /mnt/fs/file'
kubectl exec -it deployment/imagetest  -- sh -c 'cat /mnt/fs/file'
```

# 9. Volume cloning

## 9.1. Replicate a CephFS volume

Update the content on the FS volume before cloning:

```
kubectl exec -it deployment/sharedfstest -- sh -c 'echo $@ > /mnt/fs/file ; sync' sh Hello world "$(date)" '(before cloning)'
kubectl exec -it deployment/sharedfstest -- sh -c 'cat /mnt/fs/file'
```

Create a volume by cloning the original:

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc-copy2
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-filesystem
  dataSource:
    name: cephfs-pvc
    kind: PersistentVolumeClaim
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sharedfstest2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sharedfstest2
  template:
    metadata:
      labels:
        app: sharedfstest2
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - name: main
        image: ubuntu
        command: ["/bin/sh", -c, "sleep 72000"]
        volumeMounts:
        - name: fs
          mountPath: /mnt/fs
      volumes:
      - name: fs
        persistentVolumeClaim:
          claimName: cephfs-pvc-copy2
          readOnly: false
EOF
```

Update the content on the original FS volume after cloning:

```
kubectl exec -it deployment/sharedfstest -- sh -c 'echo $@ > /mnt/fs/file ; sync' sh Hello world "$(date)" '(after cloning)'
```

Check the content on the FS volume created by cloning as well as the original volume:

```
kubectl exec -it deployment/sharedfstest2 -- sh -c 'cat /mnt/fs/file'
kubectl exec -it deployment/sharedfstest  -- sh -c 'cat /mnt/fs/file'
```

## 9.2. Replicate a RBD volume

Update the content on the RBD volume before cloning:

```
kubectl exec -it deployment/imagetest -- sh -c 'echo $@ > /mnt/fs/file ; sync' sh Hello world "$(date)" '(before cloning)'
kubectl exec -it deployment/imagetest -- sh -c 'cat /mnt/fs/file'
```

Create a volume by cloning the original:

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-pvc-copy2
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-block
  dataSource:
    name: image-pvc
    kind: PersistentVolumeClaim
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: imagetest2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: imagetest2
  template:
    metadata:
      labels:
        app: imagetest2
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - name: main
        image: ubuntu
        command: ["/bin/sh", -c, "sleep 72000"]
        volumeMounts:
        - name: fs
          mountPath: /mnt/fs
      volumes:
      - name: fs
        persistentVolumeClaim:
          claimName: image-pvc-copy2
          readOnly: false
EOF
```

Update the content on the original block volume after cloning:

```
kubectl exec -it deployment/imagetest -- sh -c 'echo $@ > /mnt/fs/file ; sync' sh Hello world "$(date)" '(after cloning)'
```

Check the content on the block volume created by cloning as well as the original volume:

```
kubectl exec -it deployment/imagetest2 -- sh -c 'cat /mnt/fs/file'
kubectl exec -it deployment/imagetest  -- sh -c 'cat /mnt/fs/file'
```
