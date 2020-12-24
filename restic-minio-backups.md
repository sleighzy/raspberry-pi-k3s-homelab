# Restic MinIO Backups

[Restic] is a program that can be used to backup data to many different storage
types.

[MinIO] is open source software for object storage. This supports the S3 API so
can be used to host your very own private S3 solution. This is perfect for my
homelab and my other applications can easily integrate with this.

The instructions in this repository outline my usage of Restic and MinIO for
backing up the data on my Raspberry Pis.

## Installation and Backups

The [Restic Installation] documentation is easy to follow, as I'm running Ubuntu
on my Raspberry Pi it was just the command below.

```sh
apt-get install restic
```

MinIO is already deployed on my [K3s] cluster, see [MinIO S3] for my
instructions and manifest files for this. I will be using this as the backend
object storage for the Restic backups. The MinIO [restic with MinIO Server]
documentation is really easy to follow for the basic steps on initializing your
Restic repo to connect to S3 and then perform the backup.

Run the commands below to export the values of your Kubernetes secrets for the
MinIO access key and secret key. Note that Restic requires them to be present in
the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` environment variables.

```sh
export AWS_ACCESS_KEY_ID=$(kubectl get secrets/minio -n minio --template={{.data.MINIO_ACCESS_KEY}} | base64 --decode)
export AWS_SECRET_ACCESS_KEY=$(kubectl get secrets/minio -n minio --template={{.data.MINIO_SECRET_KEY}} | base64 --decode)
```

Next run the command below to initialize your backup repository for the first
time. The S3 bucket `restic` should be created prior to this.

```sh
$ restic -r s3:https://minio.example.com/restic init
enter password for new repository:
enter password again:
created restic repository 67df13768b at s3:https://minio.example.com/restic

Please note that knowledge of your password is required to access
the repository. Losing your password means that your data is
irrecoverably lost.
```

Use the command below to create your first backup. In the example below I'm
backing up my `/home/ubuntu/k3s` directory, which is where I keep my various
manifest files for ongoing projects.

```sh
$ restic --verbose -r s3:https://minio.example.com/restic backup /home/ubuntu/k3s
open repository
enter password for repository:
repository 79bf3a14 opened successfully, password is correct
lock repository
load index files
using parent snapshot 36e53640
start scan on [/home/ubuntu/k3s]
start backup on [/home/ubuntu/k3s]
scan finished in 0.340s: 167 files, 160.248 KiB

Files:         167 new,     0 changed,     0 unmodified
Dirs:            2 new,     0 changed,     0 unmodified
Data Blobs:     26 new
Tree Blobs:      3 new
Added to the repo: 25.342 KiB

processed 167 files, 160.248 KiB in 0:00
snapshot 6baa5a6f saved
```

Run the command below to view the lists of your snapshots over time.

```sh
$ restic --verbose -r s3:https://minio.example.com/restic snapshots
enter password for repository:
repository 79bf3a14 opened successfully, password is correct
ID        Time                 Host        Tags        Paths
-----------------------------------------------------------------------
36e53640  2020-11-21 11:27:23  k3s-1                   /home/ubuntu/k3s
6baa5a6f  2020-12-07 04:58:06  k3s-1                   /home/ubuntu/k3s
-----------------------------------------------------------------------
2 snapshots
```

[k3s]: https://k3s.io/
[minio]: https://min.io/
[minio s3]: https://github.com/sleighzy/k3s-minio-deployment
[restic]: https://restic.net/
[restic installation]:
  https://restic.readthedocs.io/en/stable/020_installation.html
[restic with minio server]: https://docs.min.io/docs/restic-with-minio.html
