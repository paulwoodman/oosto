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

## Getting Started with Git

### Creating a Git Repo with `ansible-galaxy`

Using the `ansible-galaxy` command line tool that comes bundled with Ansible, we can create the skeleton structure for a new role with the `init` command.

For example, the following will create a role directory structure called `new-repo` in the current working directory:

```
ansible-galaxy init new-repo
```

This will create the following file and directory structure:
```
new-repo/
├── README.md 
└── roles
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    ├── meta
    │   └── main.yml
    ├── tasks
    │   └── main.yml
    ├── templates
    └── vars
        └── main.yml
```

With your new Ansible role directory structure created, you can now initialize this role directory as a new Git repository. The process of making changes and saving things in a repo is called a `commit.`

In this example, we’ll initialize our new repo and make our first commit:
```
ansible-galaxy init new-repo
cd new-repo/
git init
git add .
git commit -m 'making new role'
git push
```

And if you need to modify the location of where that repo syncs to, you can change the remote push destination to a custom Git location:
```
git remote add origin <git_repo_url>
git push -u origin main
```

### Creating a Git Repo from scratch

Alternatively, in situations where the `ansible-galaxy` tool is NOT available, the following commands creates the requisite role files and will initialize the role directory as a new git repo:

```
git init new-repo

for directory in tasks handlers files templates vars defaults meta;
  do mkdir -p new-repo/roles/$directory; done

for file in tasks vars defaults meta;
  do touch new-repo/roles/$file/main.yml; done

touch new-repo/README.md

cd new-repo
git add .
git commit -m 'ansible role with all the essentials'
git push
```
