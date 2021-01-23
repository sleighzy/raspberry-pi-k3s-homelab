# Raspberry Pi K3s Homelab

This repository contains my random notes on building and deploying a [K3s]
Kubernetes cluster using Raspberry Pis. I have a short memory so this makes it
easier to remember why/how I did things...and may help others who travel down
this path.

![k3s-cluster](./raspberry-pi-4b-cluster.jpg)

## Raspberry Pi

[Raspberry Pi] are small single-board computers ideal for home projects,
clustering, and IoT edge devices. They are built using ARM processors and
provide support for numerous input and output devices.

My cluster is built with [Raspberry Pi 4 Model B]:

- Quad core ARM v8 (64bit) 1.5GHz CPU
- 8Gb RAM
- 2.45GHz and 5GHz Wi-Fi
- Bluetooth
- Gigabit ethernet
- 2 USB 2 and 2 USB 3 ports
- 2x micro-HDMI ports, supports 2 4K monitors
- Micro-SD card for operating system and internal data storage
- ...

I am running Ubuntu 20.04.1 LTS 64bit as the operating system.

## Ubuntu OS Installation

The [Install Ubuntu on a Raspberry Pi] documentation and installer is extremely
easy to follow to install the Ubuntu OS on your SD card. I chose the recommended
Ubuntu Server 20.04.1 LTS 64bit option as I was more interested in running this
headless, for now, than installing a full desktop.

## Deploying K3s

See the [K3s README file] for instructions on installing K3s on the Raspberry
Pis:

- How to deploy nodes using the standard K3s scripts
- How to deploy nodes using the awesome [k3sup] (pronounced "ketchup") tool
- Using external hard drives for storage with the K3s local-storage class
- Building images with BuildKit for containerd

## Cluster Monitoring and Grafana Loki

See the [Cluster Monitoring README file] for instructions on deploying a
monitoring solution for your Raspberry Pi K3s cluster:

- OS information for the Raspberry Pis, e.g. disk space, temperature, etc.
- Metrics on running pods and resource usage
- Grafana dashboards for viewing metrics
- Deployment of Grafana Loki for aggregating and viewing logs

## Deployed Applications

The table below contains references to my Github repositories and instructions
for installing the applications deployed in my cluster.

<!-- markdownlint-disable MD013 -->

| Application                                     | Description                                                                                                                                                                                                                                                      |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Traefik 2 Kubernetes Ingress Controller CRD]   | Manifest files, instructions, and examples for deploying the Traefik 2 Kubernetes Ingress Controller. This uses the new Kubernetes Custom Resource Definition (CRD) for Ingress Routes in Traefik 2. K3s installs Traefik 1.7 so this is a replacement for that. |
| [K3s Keycloak Deployment]                       | Deployment of [Keycloak] for authentication and authorization to applications and services.                                                                                                                                                                      |
| [OpenID Connect Traefik Forward Authentication] | Manifest files and instructions for deploying a Traefik Forward Authentication component to delegate authentication to OpenID Connect Providers such as Github or Google.                                                                                        |
| [MinIO S3]                                      | Files for deploying the Open Source [MinIO] server for S3 object storage. Contains additional documentation and policy files for integrating with Keycloak using OAuth2 / OpenID Connect to authenticate and authorize access to resources.                      |
| [Restic MinIO Backups]                          | Using [Restic] to backup data to [MinIO] S3 storage                                                                                                                                                                                                              |

<!-- markdownlint-enable MD013 -->

[cluster monitoring readme file]: monitoring-and-logging/README.md
[install ubuntu on a raspberry pi]: https://ubuntu.com/download/raspberry-pi
[k3s]: https://k3s.io/
[k3s keycloak deployment]: https://github.com/sleighzy/k3s-keycloak-deployment
[k3s readme file]: ./k3s.md
[k3sup]: https://github.com/alexellis/k3sup
[keycloak]: https://www.keycloak.org/
[minio]: https://min.io/
[minio s3]: https://github.com/sleighzy/k3s-minio-deployment
[openid connect traefik forward authentication]:
  https://github.com/sleighzy/k3s-traefik-forward-auth-openid-connect
[raspberry pi]: https://www.raspberrypi.org/
[raspberry pi 4 model b]:
  https://www.raspberrypi.org/products/raspberry-pi-4-model-b/specifications/
[restic]: https://restic.net/
[restic minio backups]: restic-minio-backups.md
[traefik 2 kubernetes ingress controller crd]:
  https://github.com/sleighzy/k3s-traefik-v2-kubernetes-crd
