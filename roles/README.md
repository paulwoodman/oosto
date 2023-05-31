# Ansible Role and Directory Structure
Our automation repository should be set up with a file structure that’s representative of our Ansible roles. This sort of directory structure allows us to easily assemble our configurations in layers.

Example Role Structure:

```
├─ wk2server
│   ├── defaults            # variable defaults that may be commonly changed
│   │   └── main.yaml
│   ├── files               # non-template files
│   │   └── aaa.cert
│   ├── handlers            # intra-role tasks
│   │   └── main.yaml
│   ├── meta                # playbook dependencies
│   │   └── main.yaml
│   ├── tasks               # playbooks
│   │   ├── service.yaml
│   │   ├── main.yaml       # determines which os to use
│   │   ├── f5-os.yaml
│   ├── templates           # jinja templates
│   │   ├── rhel_config.j2
│   └── vars                # variables and vaults
│       └── main.yaml
│       └── vault.yaml
...
├─ cleanup
├─ export_data
├─ prep4rhel8
├─ config
├─ base
```

## Ansible Role Naming Standards

When adding new roles, it’s suggested that you name them in a manner that will allow you to quickly determine playbook functions. A good naming standard will allow you to make dynamic calls to your playbooks and roles.

A few things I recommend. First, use lowercase everything; camelcase breaks Windows. Second, the only special character you should is an underscore (`_`), as YAML doesn't allow them for variables. Finally, it doesn't matter whether you use `.yaml` or `.yml`, but for sanity's sake, keep it consistent.

Here’s what Red Hat recommends for Ansible role naming standards. Galaxy roles can not have dashes in them if you are using `ansible-galaxy` to pull them.

```
  # network_config_snmp
  {{ infra }}_{{ service/func }}_{{ entity/object }}
```

Using the `network_config_snmp` example from above, it describes the service or function that it performs. 

A good naming standard opens the door to integrating inventory variables into how we call roles and files through playbooks. This adds further efficiency to our dynamic configs. Here are some examples:

```
  # config_service-rhel.yaml
  {{ service/func }}_{{ entity/object }}-{{ ansible_os }}.yaml


  # templates/ios-s2-az2.j2
  templates/{{ ansible_network_os }}-{{ site }}-{{ zone }}.j2

  # splunk/hostname-20190201-01:14:35.log
  {{ logger }}/{{ ansible_hostname }}-{{ ansible_date_time.date }}.log
```
