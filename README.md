# Raspberry Pi K3s Homelab

This repository contains my random notes on building and deploying a [K3s]
Kubernetes cluster using Raspberry Pis. I have a short memory so this makes it
easier to remember why/how I did things...and may help others who travel down
this path.

![k3s-cluster](./raspberry-pi-4b-cluster.jpg)

- [Raspberry Pi](#raspberry-pi)
  - [Ubuntu OS Installation](#ubuntu-os-installation)
    - [Using WiFi](#using-wifi)
  - [Booting from SSD](#booting-from-ssd)
    - [tl;dr](#tl-dr)
    - [Troubleshooting](#troubleshooting)
  - [Deploying K3s](#deploying-k3s)
  - [Cluster Monitoring and Grafana Loki](#cluster-monitoring-and-grafana-loki)
  - [Deployed Applications](#deployed-applications)

## Raspberry Pi

[Raspberry Pi] are small single-board computers ideal for home projects,
clustering, and IoT edge devices. They are built using ARM processors and
provide support for numerous input and output devices.

My cluster is built with three [Raspberry Pi 4 Model B]:

- Quad core ARM v8 (64bit) 1.5GHz CPU
- 8Gb RAM
- 2.45GHz and 5GHz Wi-Fi
- Bluetooth
- Gigabit ethernet
- 2 USB 2 and 2 USB 3 ports
- 2x micro-HDMI ports, supports 2 4K monitors
- Micro-SD card for operating system and internal data storage

I am running Ubuntu 20.04.2 LTS 64bit as the operating system.

Additional hardware for each:

- Kingstone A2000 1TB M.2 NVMe SSD
- Orica transparent NVMe.m2 USB-C SSD enclosure
- Yahboom RGB cooling hat with fan and lcd display

## Ubuntu OS Installation

The [Install Ubuntu on a Raspberry Pi] documentation and installer is extremely
easy to follow to install the Ubuntu OS on your SD card. I chose the recommended
Ubuntu Server 20.04.2 LTS 64bit option as I was more interested in running this
headless, for now, than installing a full desktop.

**Note:** You will need to install Raspberry Pi OS on the SD card and Ubuntu on
the SSD if you follow the instructions for booting your Pi off an SSD and no
longer needing to use an SD card.

### Using WiFi

I found that wifi did not work automatically with Ubuntu, even with the entry in
the `network-config` file. I needed to follow the instructions at
<https://linuxconfig.org/ubuntu-20-04-connect-to-wifi-from-command-line> to add
the wifi network settings to the `/etc/netplan/50-cloud-init.yaml` file.

```sh
# update the 50-cloud-init.yaml file with the wifi network settings
$ sudoedit /etc/netplan/50-cloud-init.yaml

# apply the settings
$ sudo netplan apply

# check that wifi adapter has an ip address
$ ip a
```

## Booting from SSD

SD cards are limited in capacity (in comparison to HDD/SSD with terrabytes of
space) and reported to have a short-lifetime span. An external drive can be
mounted onto the Raspberry Pi for storage, but for raw performance and removing
the dependency on the SD card for the OS an SSD drive can be used to boot and
run the Pi.

I have used an NVMe.m2 drive with a USB-C drive enclosure and Ubuntu to run the
Raspberry Pi as per a number of other blogs and guides.

The instructions on
<https://jamesachambers.com/raspberry-pi-4-ubuntu-20-04-usb-mass-storage-boot-guide/>
were excellent and easily followed. I only had a couple of items:

- I used the `stable` release channel for the EEPROM upgrade, Jame's indicates
  that this is no longer necessary but I kept for consistency across the Pis as
  it worked.
- I needed to follow the additional official Ubuntu docs to update the boot
  config so that the Pi would attempt to boot off an SSD drive if no SD card was
  found. See
  <https://ubuntu.com/tutorials/how-to-install-ubuntu-desktop-on-raspberry-pi-4#4-optional-usb-boot>
  for further information.

### tl;dr

Steps and commands I ran for future reference. Please read Jame's guide in full
as he explains each as well as the manual steps in the automated script.

- Write the Raspberry Pi OS to the SD card using the Raspberry Pi Imager
  application
- Write the Ubuntu Server 20.04.2 64bit LTS to the SSD drive, the Raspberry Pi Imager
  application will accept the SSD drive as an SD card.
- Boot the Raspberry Pi from the SD card, don't attach the SSD drive yet

Run the below commands to update the EEPROM and set bootloader config to boot
from SSD if no SD card:

```sh
# update to the "stable" channel for firmware releases
# Note: Jame's guide states that this is no longer necessary and he doesn't recommend
# edit the /etc/default/rpi-eeprom-update file and update to FIRMWARE_RELEASE_STATUS="stable"
sudo nano /etc/default/rpi-eeprom-update

# get the latest release
sudo rpi-eeprom-update

# get boot config from the latest EEPROM, this directory /stable/ matches the channel we used
# above, this may need to be /critical/ if this was left unchanged
rpi-eeprom-config --out pieeprom-new.bin --config bootconf.txt /lib/firmware/raspberrypi/bootloader/stable/pieeprom-2021-03-18.bin

# update boot config to change BOOT_ORDER from 0x1 (SD card only)
# to 0xf41 (SSD drive if no SD card)
sed -i -e '/^BOOT_ORDER=/ s/=.*$/=0xf41/' bootconf.txt

# apply new EEPROM and updated boot config
$ sudo rpi-eeprom-update -d -f ./pieeprom-new.bin
*** INSTALLING ./pieeprom-new.bin  ***
BOOTFS /boot
EEPROM update pending. Please reboot to apply the update.

# check the version
$ vcgencmd bootloader_version
Mar 18 2021 08:54:11
version 1b43d5b6fe5b71c300563afc0548122752a0618b (release)
timestamp 1616057651
update-time 1618219141
capabilities 0x0000001f

sudo reboot
```

After rebooting has completed attach the SSD drive to the USB 3 port, one of the
blue ones. Run the below commands to have the partitions setup for booting from
the SSD. Raspberry Pi does not support booting from a compressed kernel, the
steps here will automate the decompressing of the kernel and a script that is
run during upgrades to decompress any new updated kernel.

```sh
# check that the SD card and SSD drive are both mounted,
# the SD card is mmcblk0 and SSD is sda
$ lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda 8:0 0 931.5G 0 disk
├─sda1 8:1 0 256M 0 part /media/pi/system-boot
└─sda2 8:2 0 931.3G 0 part /media/pi/writable
mmcblk0 179:0 0 116.2G 0 disk
├─mmcblk0p1 179:1 0 256M 0 part /boot
└─mmcblk0p2 179:2 0 116G 0 part /

# unmount the SSD drive from the default location
sudo umount /media/pi/writable
sudo umount /media/pi/system-boot

# mount the SSD drive to the location needed for running the script to
# setup the files for booting from the decompressed kernel
sudo mkdir /mnt/boot
sudo mkdir /mnt/writable
sudo mount /dev/sda1 /mnt/boot
sudo mount /dev/sda2 /mnt/writable

# check mounts as expected
$ lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda 8:0 0 931.5G 0 disk
├─sda1 8:1 0 256M 0 part /mnt/boot
└─sda2 8:2 0 931.3G 0 part /mnt/writable
mmcblk0 179:0 0 116.2G 0 disk
├─mmcblk0p1 179:1 0 256M 0 part /boot
└─mmcblk0p2 179:2 0 116G 0 part /

# run the automated script
sudo curl https://raw.githubusercontent.com/TheRemote/Ubuntu-Server-raspi4-unofficial/master/BootFix.sh | sudo bash

# unmount the SSD drive
sudo umount /mnt/boot
sudo umount /mnt/writable

# shutdown the Raspberry Pi
sudo shutdown -h now
```

Power off the Raspberry Pi, remove the SD card and then start it back up again.
The Raspberry Pi should check for the SD card, and then after failing will boot
off the SSD drive. When you access the Raspberry Pi you should get the Ubuntu
login prompt. The default credentials are `ubuntu` / `ubuntu`.

**Note:** I needed to follow the instructions in the earlier _Using WiFi_
section of this README file to enable wifi to connect to my network. You'll be
unable to ssh to your Pi over wifi without that. You may need to use your
ethernet connection to access the Raspberry Pi initially until you have set that
up.

### Troubleshooting

If the Raspberry Pi is not booting up from the SSD drive and you are unable to
access it check for the flashing led above the SD card slot. The flashing
pattern may provide more information as to the root cause. See
<https://www.raspberrypi.org/documentation/configuration/led_blink_warnings.md>
for the list of warning flash codes. I had an issue whereby there were 4 quick
flashes, a pause, and then another repeated 4 flashes. This means
"`start*.elf not found`" which meant it couldn't locate the files for the kernel
to boot. This was not covered in the online guides but turned out to be that I
needed to update my boot config, as detailed in my instructions above, to set
the `BOOT_ORDER` to `0xf41`. With the default setting of `0x1` it was stuck
constantly retrying the SD card, which had been removed, and wasn't trying the
SSD drive.

To check to see if you have missed any steps above you will need to:

- Power off the Raspberry Pi
- Detach the SSD drive
- Insert the SD card
- Power on and log into the Raspberry Pi OS again as you have booted off the SD
  card

Once you've gone through the earlier steps and fixed (hopefully) the problem
then just repeat the steps for removing the SD card and booting up from the SSD
drive.

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
