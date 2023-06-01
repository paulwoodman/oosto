## Ansible Facts Machine

The Facts Machine parses Linux and network device configs into a data model. This role gathers native Ansible Facts or sets custom facts to parse command output and convert device configurations into code.

Once gathered, facts can be used as backups/restores, called later as variables in other roles or playbooks, and used to define the state of a device. And most importantly, facts are used to build the framework of a CMDB!

This role will is compatible with the following platforms:

```
linux
eos
ios
iosxr
nxos
aruba
aireos
f5-os
fortimgr
junos
paloalto
vyos
```

--------------

### Role Variables

Ansible uses the `ansible_network_os` inventory variable to define the device OS. This should ideally be coming from an inventory or group variable.

At a minimum, Ansible needs these details to run against hosts:
```
ansible_hostname     hostname_fqdn
ansible_network_os   ios/nxos/etc
ansible_username     username
ansible_password     password
```
