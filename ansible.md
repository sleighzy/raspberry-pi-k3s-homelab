# Ansible

[Ansible] is a provisioning and configuration automation tool that can be used
to install and config software on multiple remote hosts.

This is ideal for use with our K3s cluster as we can remotely perform operations
and install software on the Ubuntu hosts without needing to log into each
Raspberry Pi individually and run things manually.

## Creating Host Inventory

The Ansible [How to build your inventory] documentation covers everything you
need to know about creating inventory so that Ansible knows where to locate your
hosts, and the groups they belong to when running roles.

At the most basic level I have the below contents within the
`/etc/ansible/hosts` file on my Mac laptop.

```sh
k3s-1 ansible_connection=ssh ansible_user=ubuntu
k3s-2 ansible_connection=ssh ansible_user=ubuntu
k3s-3 ansible_connection=ssh ansible_user=ubuntu

[k3s-nodes]
k3s-[1:3]

[k3s-master]
k3s-1

[k3s-workers]
k3s-[2:3]
```

That file contains the following items:

- the list of hostnames for each remote host, I have mapped the ip addresses to
  the hostnames in my `/etc/hosts` file already.
- when making connections via ssh the `ubuntu` user will be the user account
  logged into on the remote host
- the `k3s-nodes` group can be used when running Ansible roles, or adhoc
  commands, to execute these on all hosts
- the `k3s-master` group can be used to run things only on the master node
- the `k3s-workers` group can be used to run things only on the worker nodes

## SSH Access

As commands are run over ssh you may need to add your public ssh key of your
local machine to each remote host for trusted password-less access. You can run
the below command against each host. You will need to provide the password for
the remote host when prompted.

```sh
ssh-copy-id ubuntu@k3s-1
```

## Pinging Hosts to Verify Connectivity

To verify that the hosts are contactable and that Ansible is able to access all
nodes in the `k3s-nodes` group the Ansible [ping] module can be used.

```sh
$ ansible k3s-nodes -m ping

k3s-2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
k3s-1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
k3s-3 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

## Running Adhoc Commands

Adhoc commands can be run against the nodes for performing single tasks. For
example, the below command will add a cron job so that a Python script is
executed whenever the node starts up.

See the Ansible [Introduction to ad hoc commands] for more information on this.

```sh
ansible k3s-nodes -u ubuntu \
  -m cron \
  -a 'name="start rpi oled display" special_time=reboot job="sudo python3 /home/ubuntu/src/github/yahboom-raspi-cooling-fan/oled.py &"'

k3s-2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "envs": [],
    "jobs": [
        "start rpi oled display"
    ]
}
k3s-1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "envs": [],
    "jobs": [
        "start rpi oled display"
    ]
}
k3s-3 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "envs": [],
    "jobs": [
        "start rpi oled display"
    ]
}
```

[ansible]: https://www.ansible.com/
[how to build your inventory]:
  https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html
[introduction to ad hoc commands]:
  https://docs.ansible.com/ansible/latest/user_guide/intro_adhoc.html
[ping]:
  https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ping_module.html
