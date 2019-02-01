# Basic Ansible Overview

Ansible is known as a configuration management tool used to automate installations and configurations. It provides much more such as application deployment, provisioning , continuous delivery and orchestration. It is a very powerful tool and is agentless - meaning it uses OpenSSH & WinRM, No updates or Agents.

Ansible typically comes with 2 files to run: Host Inventroy (ini file) and Playbook (yml file). These are usually managed on a Ansible Management node then the management node uses SSH trusts to run the playbooks to other nodes.


![ans0](https://github.com/sxcdennis/Ansible/blob/master/images/ans0.png?raw=true)



Personally, I think that Ansible is trying to solve the issue of repetitive tasks by automating everything. In this repo I will talk about what Ansible is, how to use it and show some examples of problems that we can solve with it.


### Ways to run Ansible

There are 3 ways to run Ansible:  

**Ad-Hoc:** ansible <inventroy> -m

**Playbooks:** ansible-playbook

**Automation Framework:** Ansible Tower


### Ad-Hoc Commands

Ad-hoc commands are commands that run or call a module directly from the command line. No playbook is required.

**Syntax**

```
ansible <inventory> <options>

```

**Common Options**

```
ansible web -a /bin/date
ansbile web -m ping
ansible web -m yum -a "name=openssl state=latest"

```

Commands are good and all but to do more complex and automatic tasks, it would be better with playbooks. Still better/quicker than running commands on each box though.

# Inventories

Host inventory file is a INI file that is basically a listing of hosts that you want to manage with Ansible, you can group them together under random headings too.

Static Lines of servers
Ranges
Other custom things
Dynamic Lists of servers: AWS, Azure, GCP

**EXAMPLE**

```

---

[loadbalancer]
haproxy-01.example.com
haproxy-02
192.168.101.12

[web]
web1
web2
192.168.101.150

[testing]
demo.example.com


```

For group names (load balancer, web, testing) you can name them whatever you want. For host names you can name them by their fully qualified names, hostnames or IP addresses.

# Playbooks

Playbooks are configuration files that outline tasks that are going to be performed on hosts in the host inventory. This configuration file is a YML (Yet another Markup Language) File.

Playbooks contain  **plays**
Plays contain **tasks**
Tasks call **modules**

Tasks run sequentially

**Handlers** are triggered by tasks and are run once at the end of plays.

**Play Example:**

```
---
- name: install and start apache
  hosts: web
  remote_user: dennis
  become_method: sudo
  become_user: root
  vars:
    http_port: 80
    max_clients: 200

  tasks:
  - name: install httpd
    yum: name=httpd state=latest
  - name: write apache config file
    template: src=srv/httpd.j2 dest=/etc/httpd.conf
    notify:
    -restart apache
  - name: start httpd
    service: name=httpd state=running

  handlers:
  - name: restart apache
    service: name=httpd state=restarted


```

On the above example. We start by seeing that its a YML file (---).
We see the beginning where it starts with our details on what to do. Looking at the host inventory file we call **web**. Next we look at the tasks and specifics to it. Finally finish with handlers.



### Modules

Standard structure:

```

module: directive1=value direct2=value

```


### Variables

There are many different ways to source variables:
- Playbooks
- Files
- Inventories (group vars, host vars)
- Command Line
- Discovered variables (facts)


### Ansible Roles

Ansible Roles are a special kind of Playbook that are fully self-contained with tasks, variables, configuration templates and other supporting files. This helps package and ship your Ansible files easily throughout your organization.

### Running Playbooks

To run a playbook you must use the command line.

**Syntax**

```

ansible-playbook <options>
ansible-playbook my-playbook.yml


```

Using the **-C** option will help enables check mode which validates playbooks before making changes.

## Example:

```

ansible-playbook test1.yml

```

This will run test1.yml and run each  task in order then finally using the handlers.


![ans1](https://github.com/sxcdennis/Ansible/blob/master/images/ans1.png?raw=true)


If we run the same command again, Ansbile will check to see if there are changes to the hosts and if there aren't, it will not go through with the tasks.


[< Back: Table Of Contents](https://github.com/sxcdennis/Ansible) || [Next: Basic YMAL Syntax >](https://github.com/sxcdennis/Ansible/blob/master/ymal.md)
