# VM management with virsh

With `virtinst` installed, we can view and manage VMs.

My system currently has a vm installed from a while back:
```virsh list --all
 Id   Name          State
------------------------------
 -    archlinux-2   shut off
 ```

### Basic commands

```virsh start archlinux-2
virsh console archlinux-2
virsh shutdown archlinux-2
virsh dominfo archlinux-2
virsh dumpxml archlinux-2 > archlinux-2.xml
```

### What I learned
- The `virsh` tool can manage any VM that is installed (cli and gui created).
- `virsh list` shows all defined DMs, even if they are not running.
- The `Id` field only appears when a VM is running.
- VMs use XML definitions so that they are reproducible and easy to maintain.




