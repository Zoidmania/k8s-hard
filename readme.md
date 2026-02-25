# K8s the Hard Way

My attempt following https://github.com/kelseyhightower/kubernetes-the-hard-way.

## Host

I'm virtualizing all of this with Bruce, my homelab Threadripper workstation.
It's also an exercise working with Qemu; I've known about it for a while but
never spent any time working with it since I've had access to Proxmox or
Hyper-V at $WORK.

```
$ neofetch
             /////////////                zoid@bruce
         /////////////////////            ----------
      ///////*767////////////////         OS: Pop!_OS 24.04 LTS x86_64
    //////7676767676*//////////////       Kernel: 6.18.7-76061807-generic
   /////76767//7676767//////////////      Uptime: 53 mins
  /////767676///*76767///////////////     Packages: 2210 (dpkg)
 ///////767676///76767.///7676*///////    Shell: bash 5.2.21
/////////767676//76767///767676////////   Resolution: 1024x768
//////////76767676767////76767/////////   DE: COSMIC
///////////76767676//////7676//////////   Theme: adw-gtk3-dark [GTK3]
////////////,7676,///////767///////////   Icons: Cosmic [GTK3]
/////////////*7676///////76////////////   Terminal: cosmic-term
///////////////7676////////////////////   CPU: AMD Ryzen Threadripper PRO 3995WX s (128) @ 4.311GHz
 ///////////////7676///767////////////    GPU: NVIDIA Quadro RTX 4000
  //////////////////////'////////////     Memory: 29710MiB / 257529MiB
   //////.7676767676767676767,//////
    /////767676767676767676767/////
      ///////////////////////////
         /////////////////////
             /////////////
```

I installed Qemu and `libvirt`, using Virtual Machine Manager to manage these VMs.

I created the following directories on my 4TB ZFS pool:

- `/data/isos`
- `/data/qemu-vms`

Since this tutorial expects Debian 12, I downloaded the ISO:

```bash
cd /data/isos
wget https://cdimage.debian.org/cdimage/archive/12.13.0/amd64/iso-dvd/debian-12.13.0-amd64-DVD-1.iso
```

Next, I installed some virtualization tooling and made sure I had permission to use it:

```bash
sudo apt install qemu-system libvirt libvirt-daemon-system libvirt-clients ebtables dnsmasq-base \
    libvirt-dev
sudo usermod -aG libvirt zoid
sudo usermod -aG kvm zoid
```

After a reboot, I was able to run Virtual Machine Manager.

## VMs

This course expects 4 VMs. I created them like so:

| VM Name          | Hostname | Description            | CPU | RAM   | Storage |
|------------------|----------|------------------------|-----|-------|---------|
| k8s-hard-jumpbox | jumpbox  | Administration host    | 2   | 2GB   | 20GB    |
| k8s-hard-server  | server   | Kubernetes server      | 2   | 2GB   | 20GB    |
| k8s-hard-node-0  | node-0   | Kubernetes worker node | 2   | 2GB   | 20GB    |
| k8s-hard-node-1  | node-1   | Kubernetes worker node | 2   | 2GB   | 20GB    |

I installed Debian 12 on each one like so:

- VM disk stored at `/data/qemu-vms/<vm-name>`
- domain `k8s-hard.local`
- no root password
- user `user`, password `password`
- Eastern/US timezone
- standard simple partitioning scheme
- grub installed on root disk
- no desktop environment or GNOME, select SSH server and system utilities
- use default debian mirror in the US

After the VMs boot, remove the `cdrom://` target from `/etc/apt/sources.list` on all machines.
