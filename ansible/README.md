# Ansible 

Ansible is a package that allows me to run a single task "playbook" across multiple hosts. This ensures that the same commands and configurations are run on each host, helps improve idempotence and repeatability, while also leaving an audit trail.

## Installation

`sudo apt install ansible`

Then, distribute the host public key to the guests:
```
ssh-copy-id <user>@<lab-srv1-ip>
ssh-copy-id <user>@<lab-srv2-ip>
```

## Layout

Here is a sample ansible layout:
```
linux-lab/
└── ansible/
    ├── ansible.cfg
    ├── inventory.ini
    ├── group_vars/
    │   └── lab.yml
    ├── site.yml
    └── roles/
        └── baseline/
            └── tasks/main.yml
            └── handlers/main.yml
```

### ansible.cfg

Local settings for this project that override ansible defaults. Include settings here to help make commands shorter and ensure there is consistent behavior.

Careful with comments in this file. Comments have to be on their own line.

### inventory.ini

Lists machines and defines groups. Groups allow me to tie roles to specific subsets of machines.

### lab.yml

Set variables that are shared by every guest. It is common to split a single var file up into separate files organized by topic.

Secrets go in Ansible Vault (e.g., group_vars/lab_vault.yml) and referenced like {{ vault_admin_password }}.

### site.yml

Top level playbook that specifies which roles apply to which groups. In a real production setting, it is common to have a playbooks folder with multiple playbooks defined. For example, in addition to `site.yml` there may be a `baseline.yml` and `apps.yml` to separate system configuration from app-level tasks. Here, we simply have a single playbook and multiple plays.

### tasks/main.yml

Finally, I define the tasks that the baseline role performs. This is a reusable, idempotent, system state.
Note that we prefer to use `modules` to perform tasks over `commands/shell/` for idempotence.

### handlers/main.yml

Handlers run once when there is a change to the system.

## Running ansible

With the layout setup, we can first test connectivity:

```
>> cd linux-lab/ansible
>> ansible lab -m ping

lab-srv2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
lab-srv1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}

```

Next, test applying the playbook:
`ansible-playbook site.yml --check -K`

And if there are no errors, run it for real:
`ansible-playbook site.yml -K`

The `-K` is necessary so that we provide the hosts sudoer password, unless we have configured passwordless sudo, which seems bad.

## ansible-galaxy

ansible-galaxy is a tool that can perform several role and collection related operations. First, we use it to help scaffold standard subfolders for our roles:

```
ansible-galaxy init roles/nginx
ansible-galaxy init roles/postgres
```

Again, this builds the scaffolding, but it is still up to the user to define the roles and details. Each role that is defined in `site.yml` needs a corresponding `main.yml` in `roles/<role>/tasks/main.yml`.

Another command that may be useful is:

`ansible-galaxy list`.

However, this command is used to show all roles that are installed system-wide **or** in the user's default roles directory (typically `~/.ansible/roles`). In my case for this project, this throws a few warnings and an error as it can't find any usable roles. Instead, it is possible to specify a roles path:

`ansible-galaxy list --roles-path ~/linux-lab/ansible/roles`

but this seems redundant as a simple `ls` can accomplish the same thing.


## Permissions

Running 

`ansible-playbook site.yml --check`

Can throws a fatal error:

`fatal: [lab-srv1]: FAILED! => {"msg": "Missing sudo password"}`

This does **not** mean that `ansible-playbook` should be run using sudo. Instead, there are some tasks that require sudo privileges on the guests themselves. So, this may be an indication that the user doesn't have sudo privileges. Log into the guests and confirm/fix.


## ansible-inventory

Can help quickly visualize the project:
```
>> ansible-inventory --graph
@all:
  |--@ungrouped:
  |--@lab:
  |  |--lab-srv1
  |  |--lab-srv2
  |--@web:
  |  |--lab-srv1
  |--@db:
  |  |--lab-srv2
  ```

  We see the schema as defined in `inventory.ini`. 2 hosts that are part of the lab pool; one dedicated to web services and the other to the db.

