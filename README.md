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

## Enabling Playbook Debugging and Connection Logging

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

**NOTE** 
With great power comes great responsibility -- both of these log settings above are *super* insecure. They will give you *everything* that Ansible does and sees — the deepest view of exactly what’s happening on a remote device. And you’ll expose vars/logs as they’re unencrypted and made readable. Do be careful!

When it comes to a traditional OS, setting `ansible_keep_remote_files` will allow you to see process events, system calls, and whatnot that have happened on the remote system.

And on the network and non-OS side, you can enable `ansible_persistent_log_messages` to see netconf responses, system calls, and other such things from your network hosts.

```
2021-03-01 20:35:23,127 p=26577 u=ec2-user n=ansible | jsonrpc response: {"jsonrpc": "2.0", "id": "9c4d684c-d252-4b6d-b624-dff71b20e0d3", "result": [["vvvv", "loaded netconf plugin default from path /home/ec2-user/ansible/ansible_venv/lib/python3.7/site-packages/ansible/plugins/netconf/default.py for network_os default"], ["log", "network_os is set to default"], ["warning", "Persistent connection logging is enabled for lab1-idf1-acc-sw01. This will log ALL interactions to /home/ec2-user/ansible/ansible_debug.log and WILL NOT redact sensitive configuration like passwords. USE WITH CAUTION!"]]}
```

--------------

## Scale and Performance Testing

Ansible is built around many types of plugins, and some of most useful are `Callback Plugins`. These allow you to unlock some very interesting capabilities to Ansible, such as making your computer read the playbook as it runs.

Ansible ships with a number of callback plugins that are ready to use out-of-the-box — simply add a comma separated list of callback plugins to `callback_whitelist` in your `ansible.cfg` file.

The particular callback plugin that will be most helpful with performance tuning playbooks is called `profile_tasks`. It prints out a detailed breakdown of task execution times, sorted from longest to shortest, as well as a running timer during play execution. Speaking of, `timer` is another useful callback plugin that prints total execution time with more friendly output.

Ultimately, let’s start with these. Edit your `ansible.cfg` file to enable these callback plugins:

```
vi ansible.cfg
callback_whitelist = profile_tasks, timer
```

With the `profile_tasks` and `timer` callback plugins enabled, run your playbook again and you’ll see more output. For example, here’s a profile of fact collection tasks on a single inventory host:

```
ansible-playbook facts.yml --ask-vault-pass -e "survey_hosts=cisco-ios"
```

```
ansible_facts : collect output from ios device ------------ 1.94s
ansible_facts : include cisco-ios tasks ------------------- 0.50s
ansible_facts : set config_lines fact --------------------- 0.26s
ansible_facts : set version fact -------------------------- 0.07s
ansible_facts : set management interface name fact -------- 0.07s
ansible_facts : set model number -------------------------- 0.07s
ansible_facts : set config fact --------------------------- 0.07s
```

And a profile of a `change password` task on a host:

```
ansible-playbook config_localpw.yml -e "survey_hosts=cisco-ios"
```

```
config_localpw : Update line passwords --------------------- 4.66s
ansible_facts : collect output from ios device ------------- 5.06s
ansible_facts : include cisco-ios tasks -------------------- 0.51s
config_localpw : Update line passwords --------------------- 0.34s
config_localpw : Update enable and username config lines --- 0.33s
config_localpw : debug ------------------------------------- 0.23s
config_localpw : Update enable and username config lines --- 0.22s
config_localpw : Update terminal server username doorbell -- 0.22s
config_localpw : Update line passwords --------------------- 0.20s
config_localpw : Update terminal server username doorbell -- 0.20s
config_localpw : Update terminal server username doorbell -- 0.19s
config_localpw : debug ------------------------------------- 0.19s
config_localpw : Identify if it has a modem ---------------- 0.14s
config_localpw : set_fact - Modem slot 2 ------------------- 0.11s
config_localpw : set_fact - Modem slot 1 ------------------- 0.10s
config_localpw : Update terminal server username doorbell -- 0.10s
config_localpw : set_fact - Modem slot 3 ------------------- 0.10s
config_localpw : Update line passwords --------------------- 0.10s
config_localpw : Update enable and username config lines --- 0.09s
config_localpw : Update enable and username config lines --- 0.09s
```

Automation will be unique to every playbooks, host, and organization, so it’s important to regularly track performance benchmarks as your roles evolves. Beyond the obvious benefit of being able to accurately estimate your automation run times, you can determine where improvements can be made while proactively monitoring for faulty code/logic that will inevitably slip through peer reviews.

--------------

## Ansible Process Monitoring via CLI

It's recommended to use `dstat` when you're trying to benchmark Ansible CLI performance. It's incredibly useful and informative when you want to log/monitor any and all system stats related to how Ansible runs. `dstat` is an all-in-one stat collection tool that will gather everything around CPU, process, memory, I/O, network, system load, etc…

For instance, this `dstat` command will capture all the following output and display it in a graph that updates every second:

```
dstat -tcmslrnpyi --fs --socket --unix --top-oom
   -t  timestamp
   -c total CPU usage
   -m memory usage
   -s swap space
   -l load average
   -r disk I/O
   -n network I/O
   -p processes
   -y linux system stats
   -i interrupts
   --fs filesystem open files and inodes
   --socket network sockets
   --unix unix sockets--top-oom watch for OOM process
```

**Note:** In RHEL8+, `dstat` is supplied by the `pcp-system-tools` package.

--------------

## GitHub Actions for CI/CD

Ansible CI/CD workflows are recommended to be configured via GitHub Actions.

In the top-level repo, create the following directory:
```
.github/workflows/
```

And within the GitHub workflows directory, create a config file with the name of your choice:
```
.github/workflows/dev_merge.yml
```

Populate the validate file with these details. This will check out a dev branch tagged with a version, run pre-commits and linting, and -- assuming they all pass -- those changes would then be pushed to the main branch with a new tag.

```
on:
  push:
    branches: [dev]
    # tags:
    #  - v*

jobs:
  build:

   runs-on: ubuntu-latest

   steps:
   - uses: actions/checkout@v3

   - uses: pre-commit/action@v3.0.0

   - uses: ansible-community/ansible-lint-action@main

   - uses: stefanzweifel/git-auto-commit-action@v4
     with:
      commit_message: publish release
      branch: main
      # tagging_message: 'v1.2.3'
```

With a GitHub Action setup like this, it would trigger the corresponding rules only when a version tag is present. Or we could simply add the `git-auto-commit` action to pull from dev and push to main — along with adding a version tag.

To finish this process, the last thing we need to do is set the dev branch to be a protected branch. This will require someone to approve a PR to the `dev` branch – at this point they will be able to see the full results of, and a pass/fail for, each of the steps taken by Actions:

```
Checkout
Pre-commit
Linting
Auto-commit
```
