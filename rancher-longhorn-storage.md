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
using Helm. I installed Rancher Longhorn from within the _Apps & Marketplace_
page in the dashboard. Once installed I then added my Raspberry Pi worker nodes
as Longhorn nodes and set the number of replicas to 2.

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

[install/upgrade rancher on a kubernetes cluster]:
  https://rancher.com/docs/rancher/v2.5/en/installation/install-rancher-on-k8s/
[longhorn-attached-volume]: ./assets/longhorn-attached-volume.png
[longhorn-dashboard]: ./assets/longhorn-dashboard.png
[longhorn-nodes]: ./assets/longhorn-nodes.png
[longhorn-rebuilding-volume]: ./assets/longhorn-rebuilding-volume.png
[longhorn]: https://longhorn.io/
[rancher]: https://rancher.com/products/rancher/
