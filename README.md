h3po.proxmox_cloudinit_bootstrap
=========

:warning: This role is WIP and largely untested

This ansible role creates proxmox VMs from cloudinit-capable templates, such as the ones created by [h3po.proxmox_debian_cloudimage](https://github.com/h3po/ansible-role-proxmox-debian-cloudimage/).
It can edit basic parameters of the resulting VM such as CPU cores, memory and disk size and may add a freshly created ssh key for the initial connection.

Role Variables
--------------

Required variables
| variable | default value | description/alternatives
| --- | --- | --- |
| proxmox_vm_storage | local | the storage that the VM is cloned onto
| proxmox_bootstrap_user | bootstrap | the name of the user that is added to the VM
| proxmox_bootstrap_sshkey | random | the ssh key of the bootstrap user. if this is "random", a new key will be created for the first connection
| proxmox_cloudimage_template_vmid | | the id of the template to be cloned, usually set by the proxmox_debian_cloudimage role

Optional variables
| variable | default value | description/alternatives
| --- | --- | --- |
| proxmox_vm_pool | | the pool that the VM is added to
| proxmox_vm_node | | the node that the VM is cloned to. Only works when proxmox_vm_storage is a shared storage
| proxmox_cores_spec | | the number of cores
| proxmox_cputype_spec | | the KVM cpu model
| proxmox_cpuunits_spec | | the cpu shares
| proxmox_memory_spec | | the memory size in MB
| proxmox_balloon_spec | | the minimum memory size in MB with ballooning
| proxmox_memory_shares_spec | | the memory shares for ballooning
| proxmox_bridge_device_spec | | the bridge for the net0 interface
| proxmox_bridge_device_ipaddr | | the ip address for the net0 interface
| proxmox_bridge_device_gw | | the default gateway for the net0 interface
| proxmox_bridge_vlan_spec | | the vlan tag for the net0 interface
| proxmox_disk_spec | | the disk size in proxmox notation, e.g. '10G'. The role will grow the root partition and filesystem to fill the enlarged disk
| proxmox_nameserver | | DNS server IP address for the VM.
| proxmox_searchdomain | | DNS search domain for the VM.

If the optional variables are not defined, then no change is made to the VM and it will keep the inherited settings of the template.

Example Playbook
----------------

Here's an example including the [h3po.proxmox_debian_cloudimage](https://github.com/h3po/ansible-role-proxmox-debian-cloudimage/) role to create the template and then clone it to 'new-debian-vm'. The detection of wether the VM already exists or not depends on the variable `proxmox_vmid` set by the proxmox inventory plugin from the [community.general](https://github.com/ansible-collections/community.general) collection.

```yaml
---
- name: create a debian vm
  hosts: new-debian-vm
  become: true
  gather_facts: false
  vars:
    proxmox_template_pool: template
    proxmox_cloudimage_repo_subdir: bullseye
    proxmox_cloudimage_keep: true
    proxmox_cloudimage_qemuagent: true
    proxmox_bootstrap_user: test
    proxmox_bootstrap_sshkey: ssh-ed25519 AAAAxyzkkMN test
    proxmox_cloudimage_template_node: pve1  # the node that the template is on
  tasks:

    - block:

        # the proxmox_debian_cloudimage role needs the virt-customize
        # command to add qemu-guest-agent to the template VM
        - name: install libguestfs-tools on the proxmox host
          package:
            name: libguestfs-tools
            state: present
          delegate_to: "{{ proxmox_cloudimage_template_node }}"
          become: true

        - name: create the debian template
          import_role:
            name: h3po.proxmox_debian_cloudimage
          delegate_to: "{{ proxmox_cloudimage_template_node }}"
          become: true

        - name: bootstrap the vm
          import_role:
            name: h3po.proxmox_cloudinit_bootstrap
          delegate_to: "{{ proxmox_cloudimage_template_node }}"
          become: true
      
      # this is a variable defined by the proxmox inventory plugin if the vm exists
      when: proxmox_vmid is not defined

    # the vm should now exist, but it may not be started if it wasn't just created
    - name: gather facts
      setup:

    # the debian cloud image does something on first boot that keeps the dpkg
    # cache locked for some time, we need to wait for the lock before ansible
    # can install things
    - name: wait for dpkg lock
      command: fuser /var/lib/dpkg/lock-frontend
      register: tmp
      until: tmp.rc != 0
      ignore_errors: true
      delay: 5
      retries: 60

    # example task
    - name: install updates
      ansible.builtin.apt:
        upgrade: full

    # if you're using the "random" ssh key feature, you'll need to add tasks here
    # to create your permanent user and maybe also delete the bootstrap user
```

License
-------

[GNU General Public License v3.0](LICENSE)
