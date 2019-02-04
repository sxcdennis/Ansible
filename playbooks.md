# Playbooks

**Note:** Most of this information was taken out of the Ansible Documentation and put together to be read a bit easier.

Playbooks are configuration files that outline tasks that are going to be performed on hosts in the host inventory. This configuration file is a YML (Yet another Markup Language) File.

Playbooks contain  **plays**
Plays contain **tasks**
Tasks call **modules**

Tasks run sequentially

**Handlers** are triggered by tasks and are run once at the end of plays.


# Plays

As stated before plays contain **tasks** and tasks call **modules**
Here is an example of a play.

```

- hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
  remote_user: root
  tasks:
  - name: ensure apache is at the latest version
    apt:
      name: httpd
      state: latest
  - name: write the apache config file
    template:
      src: /srv/httpd.j2
      dest: /etc/httpd.conf
    notify:
    - restart apache
  - name: ensure apache is running
    service:
      name: httpd
      state: started
  handlers:
    - name: restart apache
      service:
        name: httpd
        state: restarted


```


You can have multiple plays:

```
---
- hosts: webservers
  remote_user: root

  tasks:
  - name: ensure apache is at the latest version
    yum:
      name: httpd
      state: latest
  - name: write the apache config file
    template:
      src: /srv/httpd.j2
      dest: /etc/httpd.conf

- hosts: databases
  remote_user: root

  tasks:
  - name: ensure postgresql is at the latest version
    yum:
      name: postgresql
      state: latest
  - name: ensure that postgresql is started
    service:
      name: postgresql
      state: started
```

Now that you've seen some plays we can dissect each section from start to finish (Hosts and Users, Tasks,Action Shorthand, and Handlers).

## Hosts and users

For each play in a playbook, you get to choose which machines in your infrastructure to target and what remote user to complete the steps (called tasks) as.

1. The hosts line is a list of one or more groups or host patterns, separated by colons (**:**). The **remote_user** is just the name of the user account:

```
---
- hosts: webservers
  remote_user: root

```

Remote users can also be defined per **task**:

```

---
- hosts: webservers
  remote_user: root
  tasks:
    - name: test connection
      ping:
      remote_user: yourname

```

2. Support for running things as another user:

```
---
- hosts: webservers
  remote_user: yourname
  become: yes

```

3. You can also use keyword become on a particular task instead of the whole play:

```

---
- hosts: webservers
  remote_user: yourname
  tasks:
    - service:
        name: nginx
        state: started
      become: yes
      become_method: sudo


```

4. You can also login as you, and then become a user different than root:

```

---
- hosts: webservers
  remote_user: yourname
  become: yes
  become_user: postgres

  ```


5. You can also use other privilege escalation methods, like su:

```

---
- hosts: webservers
  remote_user: yourname
  become: yes
  become_method: su

```


6. You can also control the order in which hosts are run. The default is to follow the order supplied by the inventory:

```

- hosts: all
  order: sorted
  gather_facts: False
  tasks:
    - debug:
        var: inventory_hostname

```


Possible values for order are:

**inventory:**
The default. The order is 'as provided' by the inventory

**reverse_inventory:**
As the name implies, this reverses the order 'as provided' by the inventory

**sorted:**
Hosts are alphabetically sorted by name

**reverse_sorted:**
Hosts are sorted by name in reverse alphabetical order

**shuffle:**
Hosts are randomly ordered each run



## Tasks

Each play contains a list of tasks. Tasks are executed in order, one at a time, against all machines matched by the host pattern.
When running the playbook, which runs top to bottom, hosts with failed tasks are taken out of the rotation for the entire playbook.

The goal of each task is to execute a **module**, with very specific arguments. Variables, as mentioned above, can be used in arguments to modules.

Every task should have a **name**, which is included in the output from running the playbook. The name can be anything, but usually describes the task.

**Tasks** can be declared using the legacy `action: module options` format, but it is recommended that you use the more conventional `module: options` format (plus it's less typing).

1. Here is what a basic task looks like. As with most modules, the **service module** takes `key=value` arguments:

```
tasks:
  - name: make sure apache is running
    service:
      name: httpd
      state: started

```

2. The **command and shell modules** are the only modules that just take a list of arguments and don't use the `key=value` form. This makes them work as simply as you would expect:

```

tasks:
  - name: enable selinux
    command: /sbin/setenforce 1

```

3. The command and shell module care about return codes, so if you have a command whose successful exit code is not zero, you may wish to do this:

```
tasks:
  - name: run this command and ignore the result
    shell: /usr/bin/somecommand || /bin/true

OR

tasks:
  - name: run this command and ignore the result
    shell: /usr/bin/somecommand
    ignore_errors: True

```

4. If the action line is getting too long for comfort you can break it on a space and indent(NOT TAB but two spaces) any continuation lines:

```

tasks:
  - name: Copy ansible inventory file to client
    copy: src=/etc/ansible/hosts dest=/etc/ansible/hosts
            owner=root group=root mode=0644

```


5. Variables can be used in action lines. Suppose you defined a variable called vhost in the vars section, you could do this:

```
tasks:
  - name: create a virtual host file for {{ vhost }}
    template:
      src: somefile.j2
      dest: /etc/httpd/conf.d/{{ vhost }}

```


Those same variables are usable in templates, which we'll get to later.

Now in a very basic playbook all the tasks will be listed directly in that play, though it will usually make more sense to break up tasks as described in [Roles (Reusable Playbooks)](https://github.com/sxcdennis/Ansible/blob/master/roles.md)


## Action Shorthand

Ansible prefers listing modules like this:

```
template:
    src: templates/foo.j2
    dest: /etc/foo.conf

```

Early versions of Ansible used the following format, which still works:


```

action: template src=templates/foo.j2 dest=/etc/foo.conf

```


## Handlers: Running Operations On Change

These `'notify'` actions are triggered at the end of each block of tasks in a play, and will only be triggered once even if notified by multiple different tasks.

For instance, multiple resources may indicate that apache needs to be restarted because they have changed a config file, but apache will only be bounced once to avoid unnecessary restarts.

Here's an example of restarting two services when the contents of a file change, but only if the file changes:

```

- name: template configuration file
  template:
    src: template.j2
    dest: /etc/foo.conf
  notify:
     - restart memcached
     - restart apache

```

The things listed in the notify section of a task are called **handlers**.

**Handlers** are lists of tasks, not really any different from regular tasks, that are referenced by a globally unique name, and are notified by notifiers. If nothing notifies a handler, it will not run. Regardless of how many tasks notify a handler, it will run only once, after all of the tasks complete in a particular play.

Here's an example handlers section:

```

handlers:
    - name: restart memcached
      service:
        name: memcached
        state: restarted
    - name: restart apache
      service:
        name: apache
        state: restarted

```

As of Ansible 2.2, handlers can also 'listen' to generic topics, and tasks can notify those topics as follows:

```
handlers:
    - name: restart memcached
      service:
        name: memcached
        state: restarted
      listen: "restart web services"
    - name: restart apache
      service:
        name: apache
        state:restarted
      listen: "restart web services"

tasks:
    - name: restart everything
      command: echo "this task will restart the web services"
      notify: "restart web services"

```


This use makes it much easier to trigger multiple handlers. It also decouples handlers from their names, making it easier to share handlers among playbooks and roles (especially when using 3rd party roles from a shared source like Galaxy).

Notify handlers are always run in the same order they are defined, not in the order listed in the notify-statement. This is also the case for handlers using listen.
Handler names and listen topics live in a global namespace.
If two handler tasks have the same name, only one will run.

## Executing A Playbook

To execute a playbook you use the command **ansible-playbook playbookfile**

**Example:**

```

ansible-playbook playbook.yml

```

## Ansible-Pull  

Ansible-pull pulls playbooks from a repo and executes them for the local host.

Syntax:

```
ansible-pull -U <repository> [options] [<playbook.yml>]

```

## Tips


1. To check syntax of a playbook you can use --syntax-check option

**Example:**  

```
ansible-playbook  --syntax-check test.yml

```

2. To see what hosts would be affected by a playbook before you run it:

```

ansible-playbook playbook.yml --list-hosts

```

[< Back: Creating inventory files](https://github.com/sxcdennis/Ansible/blob/master/inventory.md) || [Next: Reusable Playbooks: Includes, Imports, and Roles >](https://github.com/sxcdennis/Ansible/blob/master/roles.md)
