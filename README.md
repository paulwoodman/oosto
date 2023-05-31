# Convergint Ansible -- Upgrade RHEL and Install Oosto
Red Hat and Convergint shared repository for project code

## Getting Started with Ansible

### Inventory

At a minimum, Ansible needs an [inventory file](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html) with these details to run against hosts:
```
ansible_hostname     hostname_fqdn
ansible_username     username
ansible_password     password
```

I *highly* recommend [vaulting your passwords/keys/creds](https://docs.ansible.com/ansible/latest/user_guide/vault.html#creating-encrypted-variables) instead of storing them plaintext! My inventories usually start like this:

```
[all]
dev-anv1 ansible_host=10.0.15.14
nxos-dc1-rtr
nxos-dc2-rtr

[all:vars]
ansible_user=admin
ansible_password: !vault |
       $ANSIBLE_VAULT;1.2;AES256;ansible_user
       66386134653765386232383236303063623663343437643766386435663632343266393064373933
       3661666132363339303639353538316662616638356631650a316338316663666439383138353032
       63393934343937373637306162366265383461316334383132626462656463363630613832313562
       3837646266663835640a313164343535316666653031353763613037656362613535633538386539
       65656439626166666363323435613131643066353762333232326232323565376635

[servers]
dev-anv1 ansible_host=10.0.15.14

[server:vars]
ansible_become=yes
ansible_become_method=enable
ansible_os=redhat

[nxos]
nxos-dc1-rtr
nxos-dc2-rtr

[nxos:vars]
ansible_become=yes
ansible_become_method=enable
ansible_network_os=nxos
ansible_connection=network_cli
```

--------------

### Fact Collection and Config Parsing

Ansible's native fact gathering can be invoked by setting `gather_facts: true` in your top level playbook. And every major vendor has fact modules that you can use in a playbook task: `ios_facts`, `f5_facts`, `vmware_facts`, etc...

Just enable `gather_facts`, and you're on your way! Here's an example of gathering facts on a Cisco IOS device to create a backup of the full running config, and parse config subsets into a platform-agnostic data model:

```
- name: collect device facts
  hosts: all
  gather_facts: yes
```

Or at the task level:

```
  tasks:
  - name: gather ios facts
    ios_facts:
      gather_subset: all
```

Regardless of which you prefer, Ansible will give you all the facts about your network devices. This is how you start down the path to true Config-to-Code!

```
ansible_facts:
  ansible_net_fqdn: ios-dc2-rtr.lab.vault112
  ansible_net_gather_subset:
  - interfaces
  ansible_net_hostname: ios-dc2-rtr
  ansible_net_serialnum: X11G14CLASSIFIED...
  ansible_net_system: nxos
  ansible_net_model: 93180yc-ex
  ansible_net_version: 14.22.0F
  ansible_network_resources:
    interfaces:
    - name: Ethernet1/1
      enabled: true
      mode: trunk
    - name: Ethernet1/2
      enabled: false
    ...
```

### Caching Facts

You can also run custom commands, save the output, and parse the configuration later. Any command output can be parsed and set as an Ansible Fact! Setting custom facts and using text parsers works particularly well for building out infrastructure checks/verifications. Or perhaps if you simply want to find something that facts don't already identify:

```
- name: run command
  shell:
  commands: 
    date
  register: output

- name: set version fact
  set_fact:
    cacheable: true
    ansible_custom_date: "{{ output.stdout[0] | regex_search('Version (\S+)', '\1') | first }}"
```

Ansible Facts can be cached too! Options include local file, memcached, Redis, and a plethora of others, via Ansible's [Cache Plugins](https://docs.ansible.com/ansible/latest/plugins/cache.html).

The combination of using facts and fact caching can allow you to poll existing, in-memory data rather than parsing numerous additional commands to constantly check/refresh the device's running config.

--------------

** Playbook Debugging and Connection Logging

Ansible allows you to enable debug logging at a number of levels. Most importantly, at both the playbook run level, and the network connection level. Having that info will become invaluable with your future troubleshooting efforts.

From the CLI, you can set these as environment variables before running a playbook:

```
export ANSIBLE_DEBUG=true
export ANSIBLE_LOG_PATH=/tmp/ansible-debug.log
```

And then run your playbook with `-v`:

```
ansible-playbook … -v
-v will show increased logging around individual plays/tasks
-vvv will show Ansible playbook execution run logs
-vvvv will show SSH and TCP connection logging
```

If standard debug options aren’t enough…if you want to *truly* see everything that’s happening with Ansible, say no more! You can keep remote log files and enable persistent log messages.

```
export ANSIBLE_KEEP_REMOTE_FILES=true
export ANSIBLE_PERSISTENT_LOG_MESSAGES=True
```

**NOTE** With great power comes great responsibility -- both of these log settings above are *super* insecure. They will give you *everything* that Ansible does and sees — the deepest view of exactly what’s happening on a remote device. And you’ll expose vars/logs as they’re unencrypted and made readable. Do be careful!

When it comes to a traditional OS, setting `ansible_keep_remote_files` will allow you to see process events, system calls, and whatnot that have happened on the remote system.

And on the network and non-OS side, you can enable `ansible_persistent_log_messages` to see netconf responses, system calls, and other such things from your network hosts.

```
2021-03-01 20:35:23,127 p=26577 u=ec2-user n=ansible | jsonrpc response: {"jsonrpc": "2.0", "id": "9c4d684c-d252-4b6d-b624-dff71b20e0d3", "result": [["vvvv", "loaded netconf plugin default from path /home/ec2-user/ansible/ansible_venv/lib/python3.7/site-packages/ansible/plugins/netconf/default.py for network_os default"], ["log", "network_os is set to default"], ["warning", "Persistent connection logging is enabled for lab1-idf1-acc-sw01. This will log ALL interactions to /home/ec2-user/ansible/ansible_debug.log and WILL NOT redact sensitive configuration like passwords. USE WITH CAUTION!"]]}
```
