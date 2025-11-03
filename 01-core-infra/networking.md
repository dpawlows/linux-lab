# Networking and libvirt basics

In this system, we are installing multiple VMs inside a VM pool, and we want those VMs to be connected via a virtual network. Here. we are using libvirt's default network:
```
sudo virsh net-define /usr/share/libvirt/networks/default.xml
```
When we define and then start this network,
1. libvrt creates a virtual bridge on my host (virbr0)
2. The bridge acts like an ethernet switch that connects the VMs.
3. A background `dnsmasq` process distriubtes DHCP IP addresses in the 192.168.122.0/24 range.
4. libvirt sets up NAT rulse so the VMs can reach the outside world through the host's real network interface.
5. From the perspective of my routher, all traffic looks like it's coming from my host.

            (your real LAN)
           ┌──────────────────────────┐
           │  e.g. 192.168.1.0/24     │
           │  router = 192.168.1.1    │
           └──────────┬───────────────┘
                      │  (your host NIC)
                      │
          ┌───────────┴────────────────┐
          │ Pop!_OS Host               │
          │                            │
          │  virbr0: 192.168.122.1/24  │  ← virtual bridge
          │   ↳ runs dnsmasq (DHCP/DNS/NAT)
          │                            │
          └────┬────────────┬──────────┘
               │            │
         VM1 (lab-srv1)  VM2 (lab-srv2)
         192.168.122.2   192.168.122.3

To see the details:
```sudo virsh net-dumpxml default
ip -br addr show virbr0
sudo nft list ruleset | grep virbr0
ps aux | grep dnsmasq
```

### Libvirt connection scopes
- **System (qemu:///system)** -> Host-wide VMs, requires libvirt/kvm group membership or sudo
- **Session (qemu:///session)** -> Per-user VMs in home directory.

Can set an environment variable to default to system wide:
```
export LIBVIRT_DEFAULT_URI=qemu:///system
```

Remember, QEMU is the VM emulator and hypervisor backend. We can use it with KVM accelerration so that the VMs get near native speed.

When virt tools such as virt-install, virt-manager or virsh are used, those communicate with libvirt, and libvert delegates to QEMU.

In general, QEMU can be used in a few ways, but the general syntax is:
`<hypervisor>://[user@][host]/[scope]`

A few examples:
| Example                             | Meaning                                                                              |
| ----------------------------------- | ------------------------------------------------------------------------------------ |
| `qemu:///system`                    | Use the **QEMU driver**, connect to the **system-wide** libvirt daemon on this host. |
| `qemu:///session`                   | Use the QEMU driver, connect to the **per-user session** daemon.                     |
| `qemu+ssh://david@rocinante/system` | Connect over SSH to the *remote* machine `rocinante`, to its **system daemon**.      |
| `xen:///system`                     | Same idea, but use the **Xen** hypervisor instead of QEMU.                           |

> [!Note] The Three Slashes
> The first two `(//)` start the "authority" section of the URI (similar to https://).
> The third `/` separates the "authority" from the path. In the cases above, we use either `/system` or `/session` as the path.

#### Summary

A common command is 
```virsh -c qemu:///system list```
which says:

    "Connect to the QEMU driver, on this machine, to the system-wide libvirt daemon, and list its VMs'

