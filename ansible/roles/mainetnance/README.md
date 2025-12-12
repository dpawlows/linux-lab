Role Name
=========

A role to handle regular non-critical upgrades.

Requirements
------------



Role Variables
--------------



Dependencies
------------



Example Playbook
----------------

- name: Scheduled maintenance
  hosts: lab
  become: yes
  roles:
    - role: maintenance
      vars:
        maintenance_apt_upgrade_type: "dist"
        maintenance_allow_reboot: true


License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
