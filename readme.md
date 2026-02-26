# K8s the Hard Way

My attempt following https://github.com/Zoidmania/kubernetes-the-hard-way (forked from
https://github.com/kelseyhightower/kubernetes-the-hard-way so the contents are consistent with when
I executed this project).

# Host

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

# VM Init

This course expects 4 VMs. I created them like so:

| VM Name          | Hostname | Description            | CPU | RAM   | Storage | NIC Type | NIC Name | Address        |
|------------------|----------|------------------------|-----|-------|---------|----------|----------|----------------|
| k8s-hard-jumpbox | jumpbox  | Administration host    | 2   | 2GB   | 20GB    | Bridge   | virbr0   | 192.168.122.10 |
| k8s-hard-server  | server   | Kubernetes server      | 2   | 2GB   | 20GB    | Bridge   | virbr0   | 192.168.122.11 |
| k8s-hard-node-0  | node-0   | Kubernetes worker node | 2   | 2GB   | 20GB    | Bridge   | virbr0   | 192.168.122.12 |
| k8s-hard-node-1  | node-1   | Kubernetes worker node | 2   | 2GB   | 20GB    | Bridge   | virbr0   | 192.168.122.13 |

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

I defined static IPs on the network created by the bridge interface in each VM's
`/etc/network/interfaces` like so:

```
allow-hotplug enp1s0
iface enp1s0 inet static
    address 192.168.122.10
    netmask 255.255.255.0
    network 192.168.122.0
    broadcast 192.168.122.255
    gateway 192.168.122.1
```

I then created the ssh config included in this repo using these IPs. I installed my public key in
the `root` user on all guests so I wouldn't have to use the Virtual Machine Manager console for each
node. That lets me use my clipboard and the like from the host. I also created a new rsa keypair
that I installed on all four machines (and added to all four machines' `~/.ssh/authorized_keys`) so
each VM could reach each other VM easily.

> [!NOTE]
> TIL that Linux interface names are limited to 15 characters. I had to rename the bridge interface
> on the host to `virbr0` so the VMs could start.

Finally, I updated `/etc/hosts` in each VM so they new the static IP assignments of each other.

# Jumpbox Setup

Followed https://github.com/Zoidmania/kubernetes-the-hard-way/blob/master/docs/02-jumpbox.md.
This resulted in the following:

```
root@jumpbox:~/kubernetes-the-hard-way# tree downloads
downloads
├── client
│   ├── etcdctl
│   └── kubectl
├── cni-plugins
│   ├── bandwidth
│   ├── bridge
│   ├── dhcp
│   ├── dummy
│   ├── firewall
│   ├── host-device
│   ├── host-local
│   ├── ipvlan
│   ├── LICENSE
│   ├── loopback
│   ├── macvlan
│   ├── portmap
│   ├── ptp
│   ├── README.md
│   ├── sbr
│   ├── static
│   ├── tap
│   ├── tuning
│   ├── vlan
│   └── vrf
├── controller
│   ├── etcd
│   ├── kube-apiserver
│   ├── kube-controller-manager
│   └── kube-scheduler
├── downloads
│   ├── client
│   ├── cni-plugins
│   ├── controller
│   └── worker
└── worker
    ├── containerd
    ├── containerd-shim-runc-v2
    ├── containerd-stress
    ├── crictl
    ├── ctr
    ├── kubelet
    ├── kube-proxy
    └── runc

10 directories, 34 files
root@jumpbox:~/kubernetes-the-hard-way# kubectl version --client
Client Version: v1.32.3
Kustomize Version: v5.5.0
```

# Compute Setup

I created the `machines.txt` in this repo as directed. The suggested pod subnets are fine to me, so
I kept them.

I had already configured SSH access for the `root` user in the above section, so I skipped this
part.

I had also set the hostnames during VM initialization, so I skipped that part as well. Verification
worked just fine:

```
$ while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} hostname --fqdn
done < machines.txt

server.k8s-hard.local
node0.k8s-hard.local
node1.k8s-hard.local
```

Note that `node-0` and `node-1` became `node0` and `node1` in my config, respectively, and that I
used the domain `k8s-hard.local` instead of `kubernetes.local`. This way I keep myself honest; I
can't just copy-paste the course invocations and cheat my way through this. I have to edit
everything for my environment first.

Finally, I had also already updated `/etc/hosts` on each VM.

# PKI Infrastructure

First, I needed to update the provided `ca.conf` to use the alternate node hostnames and domain
names I had created. In this repo, the original from the course is in `ca.conf.bak`, and my modified
one is `ca.conf`.
