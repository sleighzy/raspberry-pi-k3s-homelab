# K3s

[k3s] is a lightweight, certified Kubernetes distribution, for production
workloads from Rancher Labs.

## Installation

Before installing K3s on your Raspberry Pi you need to update the configuration
to enable the container features in the kernel. For the Ubuntu 20.04 OS the
`/boot/firmware/cmdline.txt` file needs to be updated and the below items
appended to the end of the line. You will need to restart the Raspberry Pi to
pick up these changes.

```sh
cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```

There are two common ways of installing K3s.

- The <https://get.k3s.io> script from the official docs and quickstarts
- The awesome [k3sup] (pronounced "ketchup") project. This provides a single
  binary and arguments for installing K3s in a friendly manner. This also
  provides the means for installing K3s remotely over ssh and downloading the
  kubeconfig file to your local machine.

I actually used both mechanisms when I built my cluster, so was good for
comparison purposes. I had originally used the K3s script when deploying the
master, and then used k3sup later on when I bought more Raspberry Pis for agent
nodes. I've documented both approaches below.

### Installing K3s server node with script

K3s installs [Traefik], version 1.7, as the Ingress Controller, and a service
loadbalancer (klippy-lb) by default so that the cluster is ready to go as soon
as it starts up. The instructions below will be deploying a K3s cluster
_without_ the default Traefik 1.7 as we want to deploy this ourselves so that we
can use the latest Traefik v2 Kubernetes Ingress Controller installation.

The below command installs K3s using the script.

- `--write-kubeconfig-mode 644` - sets the `kubeconfig` file permissions to
  `644` so it can be read by non-root users.
- `--disable traefik` - installs K3s without deploying Traefik as the ingress
  controller. The default Traefik version is 1.7. Traefik v2 can be installed
  instead once the K3s cluster has been deployed.

```sh
$ curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable traefik

[INFO] Finding release for channel stable
[INFO] Using v1.18.9+k3s1 as release
[INFO] Downloading hash https://github.com/rancher/k3s/releases/download/v1.18.9+k3s1/sha256sum-arm64.txt
[INFO] Downloading binary https://github.com/rancher/k3s/releases/download/v1.18.9+k3s1/k3s-arm64
[INFO] Verifying binary download
[INFO] Installing k3s to /usr/local/bin/k3s
[INFO] Creating /usr/local/bin/kubectl symlink to k3s
[INFO] Creating /usr/local/bin/crictl symlink to k3s
[INFO] Creating /usr/local/bin/ctr symlink to k3s
[INFO] Creating killall script /usr/local/bin/k3s-killall.sh
[INFO] Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO] env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO] systemd: Creating service file /etc/systemd/system/k3s.service
[INFO] systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO] systemd: Starting k3s
```

Run the below command to view the k3s logs:

```sh
sudo journalctl -u k3s -f
```

### Traefik v2 Kubernetes Ingress Controller

By default K3s installs Traefik v1.7 as the Kubernetes Ingress Controller. In
the installation instructions above I have chosen to not deploy this as I am
installing the latest v2 release instead. This provides support for both the
standard Keubernetes Ingress resource as well as the Traefik IngressRoute CRD.

See my [Kubernetes Traefik Ingress Controller CRD] Github repository for
instructions and manifest files for installing this.

### K3s systemd service

K3s is run as a Systemd service. The below command can be run to see what
arguments the K3s server is being run with.

```sh
$ cat /etc/systemd/system/k3s.service

[Unit]
Description=Lightweight Kubernetes
Documentation=https://k3s.io
Wants=network-online.target
After=network-online.target

[Install]
WantedBy=multi-user.target

[Service]
Type=notify
EnvironmentFile=/etc/systemd/system/k3s.service.env
KillMode=process
Delegate=yes
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
TimeoutStartSec=0
Restart=always
RestartSec=5s
ExecStartPre=-/sbin/modprobe br_netfilter
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/k3s \
    server \
        '--write-kubeconfig-mode' \
        '644' \
        '--disable' \
        'traefik' \
```

### Installing K3s agent node with k3sup

The k3sup documentation is excellent for deploying a K3s cluster. If you haven't
already then you'll need to copy your ssh public key to the Raspberry Pi nodes
so that remote ssh commands can be run without requiring passwords. For example,
copying my local ssh key to the Raspberry Pi that will be a K3s agent.

```sh
$ ssh-copy-id ubuntu@192.168.68.117

The authenticity of host '192.168.68.117 (192.168.68.117)' can't be established.
ECDSA key fingerprint is SHA256:UcDppi+p6hM5u5sRLDjyF/hBYjLfOjiQ/aEo2SSBhq0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 2 key(s) remain to be installed -- if you are prompted now it is to install the new keys
ubuntu@192.168.68.117's password:

Number of key(s) added:        2
```

Below are the basic commands I used to install the K3s agent on another
Raspberry Pi (192.168.68.117) and join it to the cluster.

```sh
$ export SERVER_IP=192.168.68.111
$ export AGENT_IP=192.168.68.117
$ export USER=ubuntu
$ k3sup join --ip $AGENT_IP --server-ip $SERVER_IP --user $USER

Running: k3sup join
Server IP: 192.168.68.111
K102c670a600eb3efa5cf405a16fd97302a551c6e1d88f2e96ff6a1e997435c10a0::server:8cd26b0c19d028358d3362596e9e9c35
[INFO]  Finding release for channel v1.19
[INFO]  Using v1.19.4+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.19.4+k3s1/sha256sum-arm64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.19.4+k3s1/k3s-arm64
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s-agent.service → /etc/systemd/system/k3s-agent.service.
[INFO]  systemd: Starting k3s-agent
Logs: Created symlink /etc/systemd/system/multi-user.target.wants/k3s-agent.service → /etc/systemd/system/k3s-agent.service.
Output: [INFO]  Finding release for channel v1.19
[INFO]  Using v1.19.4+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.19.4+k3s1/sha256sum-arm64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.19.4+k3s1/k3s-arm64
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
[INFO]  systemd: Starting k3s-agent
```

## External Hard Drive for Persistent Storage

K3s comes with Rancher’s [Local Path Provisioner] and this enables the ability
to create persistent volume claims out of the box

To provide a much larger disk for persistent storage I connected and mounted an
external hard drive to my K3s server node. Kubernetes by default will schedule
pods to be run on nodes as it sees fit, e.g. based on available resources on
that node. When you have a single drive connected to a single node that is to be
used for persistent storage you have to ensure that pods with persistent volume
claims are scheduled onto that node. To do so you can add a specific label to
the node with the attached disk, and then use [Kubernetes node affinity] to
schedule pods onto to that node.

Run the below command to the `disktype` label with value `hdd` on the `k3s-1`
node.

```sh
kubectl label nodes k3s-1 disktype=hdd
```

Now in your deployment manifest you can specify the `nodeAffinity` stating that
the pod needs to be placed on the node that has a key of `disktype` with value
`hdd`.

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: minio
  name: minio
  labels:
    app: minio
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - hdd
  containers:
    - name: minio
      image: minio/minio:RELEASE.2020-10-28T08-16-50Z-arm64
      args: ['server', '/data']
      ports:
        - name: minio
          containerPort: 9000
      volumeMounts:
        - name: s3-pv-storage
          mountPath: /data
```

Run the below command to edit the K3s config for the local storage provisioner.

```sh
kubectl edit configmap/local-path-config -n kube-system
```

Update the config to specify that peristent volume claims on the `k3s-1` node
should be created in the `/mnt/data/storage` directory. The K3s storage
provisioner will dynamically pick up these changes.

```yaml
apiVersion: v1
data:
  config.json: |-
    {
            "nodePathMap":[
            {
                    "node":"k3s-1",
                    "paths":["/mnt/data/storage"]
            },
            {
                    "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
                    "paths":["/var/lib/rancher/k3s/storage"]
            }
            ]
    }
```

Now when persistent volume claims are created for pods using node affinity for
the `disktype=hdd` label they will be written to the external hard drive.

Refer to the K3s [storage provisioner configuration documentation] for more
information on this.

## Mount tmpfs

The below mounts an emptyDir (ephemeral storage) using tmpfs so is stored in
memory and not even written to disk. This can help prolong the SD card life as
this reduces writes to the actual disk.

```yaml
- name: etc-pihole
  mountPath: /etc/pihole
  volumes:
- name: etc-pihole
  emptyDir:
  medium: Memory
```

## Building Container Images

K3s uses [Containerd] for running containers instead of Docker, although you can
choose as an installation option to use Docker instead. Containerd is fully OCI
compliant, all that Docker (which sits on top of this offers) provides is
additional developer and build tooling experience...which equals more bloat.
Docker does not therefore need to be installed on your Raspberry Pi to run K3s.

### Building images with BuildKit for containerd

[BuildKit] can be used to build container images when not using Docker. The
process and commands for building images using `buildctl` is slightly different
than Docker, and when you start reading the docs it can be a little overwhelming
at first.

First off, BuildKit does not build and store images in a local repository like
Docker does. These are instead built to a build cache and stored internally in
BuildKit, and must then be pushed to an external registry to be able to run the
containers. This registry can be cloud-based, e.g. Docker Hub, or can just be
your own locally run registry. In my case I use Docker Hub for some images, and
I now also run the [Docker Registry] deployed within my K3s cluster (more on
that later).

You will need to download and install BuildKit from their latest releases, see
<https://github.com/moby/buildkit/releases> for further information.

The below commands can be used to provide the credentials for containerd to push
the built image to Docker Hub. The `DOCKERHUB_TOKEN` is an access token that can
be created on the _Profile > Security_ page in Docker Hub for your account.

```sh
$ export DOCKERHUB_USERNAME=username
$ export DOCKERHUB_TOKEN=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxx
$ export DOCKER_AUTH=$(echo -n "$DOCKERHUB_USERNAME:$DOCKERHUB_TOKEN" | base64)

$ cat > ~/.docker/config.json << EOF
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "$DOCKER_AUTH"
    }
  }
}
EOF
```

The below is a basic BuildKit command to build an image and have it pushed to
Docker Hub. The plaform for the image will be based on what it was built on. In
my case I am building this on my Raspberry Pi 64bit so the image OS/architecture
will be linux/arm64.

```sh
$ buildctl build \
 --frontend=dockerfile.v0 \
 --local context=. \
 --local dockerfile=. \
 --output type=image,name=docker.io/sleighzy/keycloak:11.0.2-arm64,push=true
```

## Troubleshooting

If the k3s services do not start up properly then check the logs for further
information. If you see the below output then it is likely you haven't added the
`cgroup` items referenced in the previous section. Note, the file mentioned in
the message `(/boot/cmdline.txt on a Raspberry Pi)` is not accurate for Ubuntu
20, the parameters (see above for the complete list) need to be added to the
`/boot/firmware/cmdline.txt` file.

````sh
```sh
$ sudo journalctl -u k3s -f
-- Logs begin at Wed 2020-04-01 17:23:43 UTC. --
Oct 13 07:39:49 k3s-1 k3s[2608]: http: TLS handshake error from 127.0.0.1:46994: remote error: tls: bad certificate
Oct 13 07:39:49 k3s-1 k3s[2608]: time="2020-10-13T07:39:49.922779155Z" level=info msg="Wrote kubeconfig /etc/rancher/k3s/k3s.yaml"
Oct 13 07:39:49 k3s-1 k3s[2608]: time="2020-10-13T07:39:49.922881193Z" level=info msg="Run: k3s kubectl"
Oct 13 07:39:49 k3s-1 k3s[2608]: time="2020-10-13T07:39:49.922908637Z" level=info msg="k3s is up and running"
Oct 13 07:39:49 k3s-1 k3s[2608]: time="2020-10-13T07:39:49.923368160Z" level=error msg="Failed to find memory cgroup, you may need to add \"cgroup_memory=1 cgroup_enable=memory\" to your linux cmdline (/boot/cmdline.txt on a Raspberry Pi)"
Oct 13 07:39:49 k3s-1 k3s[2608]: time="2020-10-13T07:39:49.923450846Z" level=fatal msg="failed to find memory cgroup, you may need to add \"cgroup_memory=1 cgroup_enable=memory\" to your linux cmdline (/boot/cmdline.txt on a Raspberry Pi)"
Oct 13 07:39:49 k3s-1 systemd[1]: Started Lightweight Kubernetes.
Oct 13 07:39:49 k3s-1 systemd[1]: k3s.service: Main process exited, code=exited, status=1/FAILURE
Oct 13 07:39:49 k3s-1 systemd[1]: k3s.service: Failed with result 'exit-code'.
Oct 13 07:39:53 k3s-1 systemd[1]: Stopped Lightweight Kubernetes.
````

[buildkit]: https://github.com/moby/buildkit
[containerd]: https://containerd.io/
[docker registry]: https://hub.docker.com/_/registry
[k3s]: https://k3s.io/
[k3sup]: https://github.com/alexellis/k3sup
[kubernetes node affinity]:
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
[kubernetes traefik ingress controller crd]:
  https://github.com/sleighzy/k3s-traefik-v2-kubernetes-crd
[local path provisioner]: https://rancher.com/docs/k3s/latest/en/storage/
[storage provisioner configuration documentation]:
  https://github.com/rancher/local-path-provisioner/blob/master/README.md#Configuration
[traefik]: https://traefik.io/traefik/
