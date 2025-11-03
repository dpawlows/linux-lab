# VM Creation with virt-install 

Before creating a VM, be sure to grab an OS iso to put on the VM. Here, we start with Ubuntu Server. Not everything has to be Pop.

## Storage pool

The first step is to create a storage pool for the VM disks. Here, we use -c qemu:///system to connect to the system daemon so that they run for all users instead of creating and running the VMs connected to my own session daemon.

| Type        | URI               | Runs as               | Config stored in                     | Typical use                         |
| ----------- | ----------------- | --------------------- | ------------------------------------ | ----------------------------------- |
| **Session** | `qemu:///session` | Your user             | `~/.local/share/libvirt/`            | Personal / sandboxed VMs            |
| **System**  | `qemu:///system`  | `libvirt-qemu` (root) | `/etc/libvirt/`, `/var/lib/libvirt/` | Host-wide VMs (production / shared) |


```
sudo mkdir -p /var/lib/libvirt/images/linux-lab
sudo virsh -c qemu:///system pool-define-as linux-lab dir - - - - "/var/lib/libvirt/images/linux-lab"
sudo virsh -c qemu:///system pool-autostart linux-lab
sudo virsh -c qemu:///system pool-start linux-lab
sudo virsh -c qemu:///system pool-list
```

Changing ownership of the linux-lab directory is useful for management (and may be necessary before we can start the pool):
```
sudo chown libvirt-qemu:kvm linux-lab
sudo chmod 770 linux-lab
newgrp kvn
sudo usermod -aG kvm $USER
```

When we list the pool, we get:
```
virsh -c qemu:///system pool-list
 Name          State    Autostart
-----------------------------------
 linux-lab     active   yes
 ```

## Create a storage pool

Since we have an iso that we are using to install the VMs, we may want to keep track of the iso and manage it using virt tools. So, it is common to create another pool specifically for this purpose.

```
virsh -c qemu:///system pool-define-as isos dir - - - - "$HOME/linux-lab/isos"
virsh -c qemu:///system pool-autostart isos
virsh -c qemu:///system pool-start isos
virsh -c qemu:///system pool-list

 Name        State    Autostart
---------------------------------
 isos        active   yes
 linux-lab   active   yes
```
## Default Network

Before we create the VM, we define a default network using a system template that is shipped with libvirt:

```
sudo virsh net-define /usr/share/libvirt/networks/default.xml
```

See [networking.md](networking.md) for more details.

## libvirt permissions
I tried hosting the Ubuntu Server iso in my home directory (~/linux_lab/isos). However, on Ubuntu the libvirt-qemu process is confined by an AppArmor profile that severely restricts access to the filesystem. Only a handful of directories are allowed by default:

/var/lib/libvirt/**   rwk,
/var/tmp/**           rwk,
/tmp/**               rwk,
/home/**/.virtinst/** r,

So, I am hosting the isos in `/var/lib/libvirt/boot/` instead.

## Create VM 1

The first VM will have:
- 2 CPUs
- 2 GB ram
- 20 GB of disk space
- Hostname: lab-srv1


It is possible that there are conflicts with other active networks from previous VM installs. If this is the case, remove those networks and setup the default from libvirt.

Then,

```
sudo virt-install --connect qemu:///system --name lab-srv1 --vcpus 2 --memory 3072 --disk pool=linux-lab,size=20 --cdrom /var/lib/libvirt/boot/ubuntu-24.04.3-live-server-amd64.iso --os-variant ubuntu24.04 --network network=default --graphics vnc,listen=127.0.0.1 --noautoconsole
```

sudo is required since I am connecting to the /system daemon.

In this example, we use a graphical installer through VNC. Strange things were happening when trying to go headless.

When the VM starts, we need to find the display and connect:
```
sudo virsh vncdisplay lab-srv1
vncviewer localhost:5900
```

> [!Note] vncviewer installation
> `sudo apt install tigervnc-viewer` if it isn't installed.

## Reboot

After installation, we need to reboot. With the VM, we can do that with the following. First list all block devices attached to the VM.

```
sudo virsh --connect qemu:///system domblklist lab-srv1
 Target   Source
----------------------------------------------------------------------
 vda      /var/lib/libvirt/images/linux-lab/lab-srv1.qcow2
 sda      /var/lib/libvirt/boot/ubuntu-24.04.3-live-server-amd64.iso
```

Then shutdown and eject the ROM:

```
sudo virsh --connect qemu:///system shutdown lab-srv1
sudo virsh --connect qemu:///system destroy lab-srv1
sudo virsh --connect qemu:///system detach-disk lab-srv1 sda --config
```

Finally, start the VM:
```
sudo virsh --connect qemu:///system start lab-srv1
```
and confirm
```
sudo virsh --connect qemu:///system list --all
```

If running, it is ready to be used. First, get the server's ip. Then ping to make sure it can be reached. Finally ssh in:
```
sudo virsh --connect qemu:///system domifaddr lab-srv1
ping 192.168.122.209

ssh username@192.168.122.209

or 

virt-viewer -c qemu:///system lab-srv2
```

Lastly, verify the system:
```
hostnamectl
ip a
systemctl status ssh
```

## Additional VM installs

Simply re-run the install command above. Change the VM name and any arguments as necessary.

## Troubleshooting

It took a few attempts to get the inital VM creation to work. One of the hangups is when it fails after a certain point, the VM is created, but the OS is not installed. It may be helpful to destroy and undefine the VM:

```
sudo virsh --connect qemu:///system destroy lab-srv1
sudo virsh --connect qemu:///system undefine lab-srv1 --remove-all-storage --managed-save --snapshots-metadata --nvram
```

