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

[rhel:vars]
ansible_become=yes
ansible_become_method=enable

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

Ansible Facts can be cached too! Options include local file, memcached, Redis, and a plethora of others, via Ansible's [Cache Plugins](https://docs.ansible.com/ansible/latest/plugins/cache.html). And caching can be enabled with just the click button [in AWX and Tower](https://docs.ansible.com/ansible-tower/latest/html/userguide/job_templates.html#benefits-of-fact-caching), where you can then view facts via UI and API both.

![fact cache](https://docs.ansible.com/ansible-tower/latest/html/userguide/_images/job-templates-options-use-factcache.png)

The combination of using facts and fact caching can allow you to poll existing, in-memory data rather than parsing numerous additional commands to constantly check/refresh the device's running config.

--------------

### Configs, Commands, and Templates

The bread and butter of Ansible's simplicity is being able to quickly and easily generate configs, perform diffs, and send commands. If you already have full/partial config templates, it's nearly effortless to extract the things that can be easily converted to variables, and run the bulk of your commands through Jinja templates.

```
vlan {{ vlan_id }}
  name {{ vlan_description }}
interface port-channel66.{{ vlan_id }}
  description {{ interface_description }}
  encapsulation dot1q {{ vlan_id }}

```

Once your vars and templates are setup, you can determine where you want the config output staged. At that point, you're ready to generate a template and push commands!
