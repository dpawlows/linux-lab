# Core Infrastructure

For this project, I will be simulating a distributed linux network using a core system and using several VMs. The core system is running Pop!_OS, simply because I like the distribution and the lessons learned here are directly applicable to Ubuntu Server, and really any linux distribution in general, though specific package manager and packages may differ depending on the OS.

Before we start, we make sure that the system is configured to handle virtualization. `lscpu` provides information about the system's architecture. 

`lscpu | grep Virtualization`

    Virtualization:                          AMD-V

## Base virtualization stack

We will be running multiple VMs on the core system. In order to do this effectively, there are a few layers of software that are used:

| Package | Purpose | Notes |
|----------|----------|-------|
| **qemu-kvm** | Core hypervisor engine that runs the virtual CPUs, memory, disks and hardware emulation. | Provides near-native performance by running VMs directly on the CPU instead of full software emulation. |
| **libvirt-daemon-system** | Background service that manages VMs, networks, and storage pools. | Runs the `libvirtd` daemon that other tools (like `virsh` and `virt-manager`) communicate with. |
| **libvirt-clients** | Command-line utilities for interacting with the libvirt service. | Provides commands like `virsh list`, `virsh start`, etc. |
| **bridge-utils** | Network management tools for creating and controlling Ethernet bridges. | Lets the host act like a virtual network switch so your VMs can share your network connection. |
| **virtinst** | Command-line tools for creating and managing the VMs. | Provides the `virt-install` command, which automates VM creation without a GUI. |

We install with apt:
```
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst
```

And we start by enabling libvirtd:
`sudo systemctl enable --now libvirtd`

Next, we create a libvirt group, and add my user to it so that I can manage the VMs:

newgrp libvirt
sudo usermod -aG libvirt $USER

Finally, confirm kvm is operational:
```
sudo kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

and we can test that there aren't any issues with `virsh list --all` which lists all installed VMs.



