# Rancher Longhorn Storage

[Longhorn] is a lightweight, reliable and easy-to-use distributed block storage
system for Kubernetes.

Longhorn is free, open source software. Originally developed by Rancher Labs, it
is now being developed as a sandbox project of the Cloud Native Computing
Foundation. It can be installed on any Kubernetes cluster with Helm, with
kubectl, or with the Rancher UI.

After upgrading my Raspberry Pi cluster storage by attaching SSD drives I
migrated my Kubernetes persistent storage from the default Rancher Local Storage
provisioner that comes with K3s to the new Rancher Longhorn storage provisioner.

I use [Rancher] to manage my K3s cluster. This can be easily installed into the
K3s cluster using Helm as per their [Install/Upgrade Rancher on a Kubernetes
Cluster] documentation. The Rancher cluster management dashboard feels like
cheating after using `kubectl` for everything.

## Installing Longhorn

Rancher provides the means to install applications, this is performed internally
using Helm. I have installed Rancher Longhorn from within the _Apps &
Marketplace_ page in the Rancher dashboard. I have installed using the Helm
chart directly in the past as well.

### Only running on agent nodes

If you only want replicas running on your agent nodes (data planes) and not your
server nodes (control planes) then set the number of replicas to the same
number, or less, as your agent nodes in your cluster. This is the
`persistence.defaultClassReplicaCount` value for the Helm chart. Setting the
number of replicas to `2` for example in the Longhorn admin screen settings
after installing Longhorn will only take effect for volumes created through the
user interface. For volumes created via `kubectl` this will use the number of
replicas for the `longhorn` storage class, this will have been initially set by
the Helm chart and cannot be modified unless installing/upgrading with the Helm
chart and setting the `persistence.defaultClassReplicaCount` value. See below
for the `numberOfReplicas` parameter for the storage class after it has been
created.

```plain
$ kubectl describe storageclass longhorn

Name:            longhorn
IsDefaultClass:  Yes
Annotations:     longhorn.io/last-applied-configmap=kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: longhorn
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: "Delete"
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
  fromBackup: ""
  fsType: "ext4"
,storageclass.kubernetes.io/is-default-class=true
Provisioner:           driver.longhorn.io
Parameters:            fromBackup=,fsType=ext4,numberOfReplicas=2,staleReplicaTimeout=30
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>
```

You will also need to set the server node to **Disabled** in the Longhorn
dashboard so that volumes are not scheduled and created on it. Doing this will
ensure that you are not adding extra workload to the server nodes as they will
not be running a Longhorn container per volume and heavy I/O to persist data.
See the Longhorn documentation [Evicting Replicas on Disabled Disks or Nodes]
for further information.

![longhorn-disable-node]

**Note:** Don't worry if you have already installed Longhorn and didn't set the
default number of replicas for the storage class. I was able to achieve the
above using `helm upgrade` and the
`--set persistence.defaultClassReplicaCount=2` argument as initially when I
first installed Longhorn I hadn't set the default number of replicas to 2 (I
have a 3 node cluster). The existing volumes still had all the replicas running,
including the `k3s-1` server node, even after disabling scheduling on the
`k3s-1` server node. Using the Longhorn dashboard I updated the number of
replicas to 2 for each individual volume and this automatically removed the
`k3s-1` node.

**TODO:** Try scaling the replicas up to 3 or higher for a test volume and see
if these are spread across the nodes, e.g. 2 replicas placed on `k3s-2` and 1
replica placed on `k3s-3`.

### Update Default Storage Class

Run the below commands to list the available storage classes, and then set the
previous default `local-path` provisioner to `false` so that the Longhorn
provisioner is the default storage class used when creating persistent volumes.

<https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/>

```sh
$ kubectl get storageclass -A

NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  182d
longhorn (default)     driver.longhorn.io      Retain          Immediate              true                   14m

$ kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
storageclass.storage.k8s.io/local-path patched

$ kubectl get storageclass -A
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
longhorn (default)   driver.longhorn.io      Retain          Immediate              true                   15m
local-path           rancher.io/local-path   Delete          WaitForFirstConsumer   false                  182d
```

## Longhorn Dashboard

The Longhorn dashboard displays the nodes, their status, available disk space,
and whether or not storage can be scheduled onto them.

![longhorn-dashboard]

![longhorn-nodes]

## Accessing Physical Disk

When the persistent volume is attached to a node when the pod starts running it
will be mounted onto the hosted and the file system created. At this point you
can access the disk on the filesystem if you want to view or modify the
contents. For example, access the persistent volume claim in the Longhorn
dashboard first so that you can easily see the pvc name and the node that this
is mounted on.

![longhorn-attached-volume]

The below commands can then be run on the host that the volume has been mounted
onto to display the mount location and the contents.

```sh
$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
...
sda      8:0    0 931.5G  0 disk
├─sda1   8:1    0   256M  0 part /boot/firmware
└─sda2   8:2    0 931.3G  0 part /
sdc      8:32   0    10M  0 disk /var/lib/kubelet/pods/826b1caa-a127-41c0-b47d-71634f0a9690/volumes/kubernetes.io~csi/pvc-f5eb3c84-4bf7-4920-a936-c5dab357a62c/mount

$ sudo ls -l /var/lib/kubelet/pods/826b1caa-a127-41c0-b47d-71634f0a9690/volumes/kubernetes.io~csi/pvc-f5eb3c84-4bf7-4920-a936-c5dab357a62c/mount
total 72
-rw-rw---- 1 nobody nogroup 60724 Apr 21 09:32 acme.json
drwxrws--- 2 root   nogroup 12288 Apr 21 09:29 lost+found
```

## Using Sub Paths

When the volume is mounted and the file system created it will also create the
`lost+found` directory. Some containers do not like the root of the mount point
being used to store data. For example, the official PostgreSQL image will not
start due to the below error:

```sh
Data page checksums are disabled.
initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
It contains a lost+found directory, perhaps due to it being a mount point.
Using a mount point directly as the data directory is not recommended.
Create a subdirectory under the mount point.
```

The solution to this is to mount the volume using a sub-path for the deployment,
e.g. `postgres-data` in the example below.

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresdb-pvc
  namespace: postgres
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10G
---
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: postgres
  name: postgres
  labels:
    app: postgres

spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:13
          ports:
            - name: postgres
              containerPort: 5432
          volumeMounts:
            - name: postgresdb
              mountPath: /var/lib/postgresql/data
              subPath: postgres-data
          envFrom:
            - configMapRef:
                name: postgres-config
      volumes:
        - name: postgresdb
          persistentVolumeClaim:
            claimName: postgresdb-pvc
```

## Rebuilding Volume

I have occasionally seen the dashboard display a volume as being "degraded" and
show the rebuilding status when hovering over it. This may have been when a new
container was starting on another node. Will updated these notes further if this
occurs regularly.

![longhorn-rebuilding-volume]

## Troubleshooting

### Orphaned pods unable to be removed

If the Kubernetes cluster is not shutdown cleanly, e.g. the power is turned off
on the node without shutting down K3s first, then the pods may become orphaned
when the cluster is restarted. Kubernetes should be able to clean these up, but
there can be cases due to volume issues that it cannot. When this happens, the
logs will be spammed with the below error message when starting up the `k3s` or
`k3s-agent` services.

```plain
$ sudo journalctl -f -u k3s

kubelet_volumes.go:225] "There were many similar errors. Turn up verbosity to see them." err="orphaned pod \"97c98a18-b7d0-4dd3-bd9e-d98933c987cf\" found, but error not a directory occurred when trying to remove the volumes dir" numErrs=2
```

To resolve this issue you need to remove the files within this volume from the
disk on the node. In my experience so far these have only contained the
`vol_data.json` file.

```plain
# Find the directory that contains the volume using the pod id on the same node as the error log
$ sudo ls -l /var/lib/kubelet/pods/97c98a18-b7d0-4dd3-bd9e-d98933c987cf/volumes/kubernetes.io~csi/
drwxr-x--- 2 root root 4096 Nov  3 06:56 pvc-eb653e02-8963-4739-86e5-c11a4db5675d

# Check the contents of the directory for the file Kubernetes is unable to remove
$ sudo ls -l /var/lib/kubelet/pods/97c98a18-b7d0-4dd3-bd9e-d98933c987cf/volumes/kubernetes.io~csi/pvc-eb653e02-8963-4739-86e5-c11a4db5675d
-rw-r--r-- 1 root root 289 Nov  3 06:56 vol_data.json

# Remove this file from the disk
$ sudo rm /var/lib/kubelet/pods/97c98a18-b7d0-4dd3-bd9e-d98933c987cf/volumes/kubernetes.io~csi/
pvc-eb653e02-8963-4739-86e5-c11a4db5675d/vol_data.json
```

When you check the logs again you will see the volume has been removed. The logs
may show that another pod has failed for a similar reason so you will need to
clean that up as well.

```plain
$ sudo journalctl -f -u k3s
kubelet_volumes.go:140] "Cleaned up orphaned pod volumes dir" podUID=97c98a18-b7d0-4dd3-bd9e-d98933c987cf path="/var/lib/kubelet/pods/97c98a18-b7d0-4dd3-bd9e-d98933c987cf/volumes"
```

Check the other server and agent nodes for the same issue and repeat this
process if needed.

[evicting replicas on disabled disks or nodes]:
  https://longhorn.io/docs/1.2.3/volumes-and-nodes/disks-or-nodes-eviction/
[install/upgrade rancher on a kubernetes cluster]:
  https://rancher.com/docs/rancher/v2.5/en/installation/install-rancher-on-k8s/
[longhorn-attached-volume]: ./assets/longhorn-attached-volume.png
[longhorn-dashboard]: ./assets/longhorn-dashboard.png
[longhorn-disable-node]: ./assets/longhorn-disable-node.png
[longhorn-nodes]: ./assets/longhorn-nodes.png
[longhorn-rebuilding-volume]: ./assets/longhorn-rebuilding-volume.png
[longhorn]: https://longhorn.io/
[rancher]: https://rancher.com/products/rancher/
