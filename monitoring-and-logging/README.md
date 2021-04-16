# Cluster Monitoring and Grafana Loki

Monitoring of your running applications and the resource usage of your Raspberry
Pi cluster is very important. This includes the aggregation and display of your
logs to ensure your pods are spamming the logs with warnings and errors due to
unnoticed issues.

## Cluster Monitoring

The [Cluster Monitoring] Github repository of Carlos Eduardo's contains an easy
to use monitoring with K3s and ARM64 support to deploy the following components
into your cluster:

- Prometheus
- Prometheus Operator
- Grafana
- kube-state-metrics
- Node Exporter
- Alertmanager

Grafana includes default dashboards for monitored applications and services,
including optional ones such as Traefik.

I won't bother documenting the installation here, I just followed the
[Quickstart for K3s] guide and it deployed as expected.

**Warning:** When I initially deployed this I was using a single Raspberry Pi 4b
8Gb. I found that everything deployed fine except for Prometheus. The Prometheus
pod causes the CPU to spike until the it crashed or I removed the Prometheus pod
and deployment. This problem went away after I included the second Raspberry Pi
into the cluster. Potentially purely resource related, I haven't revisited this,
but something to be aware of.

Run the command below to port forward to the Grafana `http` service port.

```sh
kubectl port-forward -n monitoring --address 0.0.0.0 service/grafana 3000:http
```

Grafana will now be accessible by navigating to <http://k3s-server-ip:3000>

### Cluster Monitoring Grafana Dashboard

Import the [cluster-monitoring-dashboard.json] file for the Kubernetes Cluster
Monitoring Grafana dashboard. This is a modification on the default one and
displays all nodes, their OS, Kubernetes version, and updated stats for disk
space usage.

![grafana-dashboard-1] ![grafana-dashboard-2]

## Grafana Loki Log Aggregation

[Grafana Loki] is a aggregation system inspired by Prometheus. It is designed to
be very cost effective and easy to operate. It does not index the contents of
the logs, but rather a set of labels for each log stream. Using `kubectl logs`
is not a scalable solution for checking the health of your pods and looking to
see if you getting logs spammed with errors and warnings. Loki is an amazing
tool for ingesting logs from your services and providing the means for
aggregating and querying them. The data from the logs can also be used for
generating metrics, e.g. request rates and latencies from access logs.

Run the below commands to install Loki using Helm.

```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install loki --namespace=loki-stack grafana/loki-stack
```

The Loki logs can be viewed within Grafana using the _Explore_ screen.

![loki-dashboard]

### Updating Loki Configuration

The `loki.yaml` configuration file is stored in a Kubernetes secret. Run the
below command to get the configuration and pipe it to a file.

```sh
kubectl get secrets/loki -n loki-stack -o "jsonpath={.data['loki\.yaml']}" | base64 --decode > loki.yaml
```

Update the file and run the below command to update the configuration in the
secret.

```sh
$ kubectl create secret -n loki-stack generic loki \
    --from-file=loki.yaml \
    --dry-run=client \
    -o yaml \
    | kubectl apply -f -

secret/loki created
```

Configuration files are stored in base64 encoded format and there is no easy
helper for editing them. The above command uses a trick using the output of a
dry-run from the `kubectl` command which outputs this in yaml format. This is
then piped into the actual `kubectl apply` command.

Credits go to David Dooling on
<https://blog.atomist.com/updating-a-kubernetes-secret-or-configmap/> for this
tip.

### Storing Logs in Object Storage (MinIO)

Loki supports storing logs and indexes in object storage, e.g. AWS S3. I have
deployed [MinIO] into my K3s cluster as a self hosted object storage solution,
see [K3s MinIO Deployment] for instructions on this.

Update the configuration in the file to specify `s3` as the data store, provide
the URL to the MinIO server, and authentication credentials. The `loki` bucket
specified in the example url below needs to be created prior to applying and
redeploying this.

The configuration for the S3 connection for MinIO is slightly different than
using AWS S3. This must match the following to prevent connection errors and the
pod to loop endlessly with errors, see the example configuration below.

- use `https` instead of `s3` for the protocol at the start of the url
- any forward slashes in the username or password in the url must be escaped by
  using `%2F` instead
- the host name must end with a period (`.`) as this forces it to be used as an
  actual hostname and not an AWS region

**Warning:** I highly recommend tailing the logs for the Loki application when
updating configuration and redeploying the pod. If the authentication and
connection fails then the pod will go into an endless loop retrying the
connection. The pod will remain in a terminating state even after deleting it,
and the endless loop with auth errors will spam the logs and the CPU will spike.
I found I needed to restart the K3s agent on the node to reset this.

```sh
kubectl logs -f -n loki-stack -l app=loki
```

```yaml
---
compactor:
  shared_store: s3
  working_directory: /data/loki/boltdb-shipper-compactor
---
schema_config:
  configs:
    - from: '2020-10-24'
      index:
        period: 24h
        prefix: index_
      object_store: s3
      schema: v11
      store: boltdb-shipper
---
storage_config:
  boltdb_shipper:
    active_index_directory: /data/loki/boltdb-shipper-active
    cache_location: /data/loki/boltdb-shipper-cache
    cache_ttl: 24h
    shared_store: s3
  aws:
    s3: https://XXXXXXX:xxxXXX%2FXXXX%2FxxxxXXXX@minio.example.com./loki
    s3forcepathstyle: true
```

[cluster monitoring]: https://github.com/carlosedp/cluster-monitoring
[cluster-monitoring-dashboard.json]: ./cluster-monitoring-dashboard.json
[grafana loki]: https://grafana.com/oss/loki
[grafana-dashboard-1]: ./grafana-dashboard-1.png
[grafana-dashboard-2]: ./grafana-dashboard-2.png
[loki-dashboard]: ./loki-dashboard.png
[minio]: https://min.io/
[k3s minio deployment]: https://github.com/sleighzy/k3s-minio-deployment
[quickstart for k3s]:
  https://github.com/carlosedp/cluster-monitoring#quickstart-for-k3s
