# Reusable Playbooks: Includes, Imports, and Roles

**Note:** Most of this information was taken out of the Ansible Documentation and put together to be read a bit easier.

While it is possible to write a playbook in one very large file, eventually you'll want to reuse files and start to organize things. There are three ways to do this: **includes, imports, and roles**.


**Includes** and **imports** allow users to break up large playbooks into smaller files, which can be used across multiple parent playbooks or even multiple times within the same Playbook.

## Dynamic and Static

If you use any `import` Task (`import_playbook`, `import_tasks`, etc.), it will be **static**.

If you use any `include` Task (`include_tasks`, `include_role`, etc.), it will be **dynamic**.


# Includes vs. Imports

All `import` statements are pre-processed at the time playbooks are parsed.

All `include` statements are processed as they are encountered during the execution of the playbook.



### Importing Playbooks

1. It is possible to include playbooks inside a master playbook.

For example:

```

- import_playbook: webservers.yml
- import_playbook: databases.yml

```

The plays and tasks in each playbook listed will be run in the order they are listed, just as if they had been defined here directly.


### Including and Importing Task Files

1. Breaking tasks up into different files is an excellent way to organize complex sets of tasks or reuse them. A task file simply contains a flat list of tasks:

```

# common_tasks.yml
- name: placeholder foo
  command: /bin/foo
- name: placeholder bar
  command: /bin/bar

```

2. You can then use `import_tasks` or `include_tasks` to execute the tasks in a file in the main task list:

```

tasks:
- import_tasks: common_tasks.yml
# or
- include_tasks: common_tasks.yml

```

3. You can also pass variables into imports and includes:

```
tasks:
- import_tasks: wordpress.yml
  vars:
    wp_user: timmy
- import_tasks: wordpress.yml
  vars:
    wp_user: alice
- import_tasks: wordpress.yml
  vars:
    wp_user: bob

```

Static and dynamic can be mixed, however this is not recommended as it may lead to difficult-to-diagnose bugs in your playbooks.

4. Includes and imports can also be used in the `handlers`: section. For instance, if you want to define how to restart Apache, you only have to do that once for all of your playbooks. You might make a **handlers.yml** that looks like:


**handelers.yml**

```

# more_handlers.yml
- name: restart apache
  service: name=apache state=restarted

```

And in your main playbook file:

```
handlers:
- include_tasks: more_handlers.yml
# or
- import_tasks: more_handlers.yml
Note

```

You can mix in includes along with your regular non-included tasks and handlers.


# Roles

Roles are ways of automatically loading certain vars_files, tasks, and handlers based on a known file structure. Grouping content by roles also allows easy sharing of roles with other users.

## Role Directory Structure

**Example Structure**

```

site.yml
webservers.yml
fooservers.yml
roles/
   common/
     tasks/
     handlers/
     files/
     templates/
     vars/
     defaults/
     meta/
   webservers/
     tasks/
     defaults/
     meta/

```

Roles expect files to be in certain directory names. Roles must include at least one of these directories, however it is perfectly fine to exclude any which are not being used. When in use, each directory must contain a `main.yml` file, which contains the relevant content:


- `tasks` - contains the main list of tasks to be executed by the role.

- `handlers` - contains handlers, which may be used by this role or even anywhere outside this role.

- `defaults` - default variables for the role

- `vars` - other variables for the role

- `files` - contains files which can be deployed via this role.
templates - contains templates which can be deployed via this role.

- `meta` - defines some meta data for this role. See below for more details.

Other YAML files may be included in certain directories. For example, it is common practice to have platform-specific tasks included from the tasks/main.yml file:

```
# roles/example/tasks/main.yml
- name: added in 2.4, previously you used 'include'
  import_tasks: redhat.yml
  when: ansible_facts['os_family']|lower == 'redhat'
- import_tasks: debian.yml
  when: ansible_facts['os_family']|lower == 'debian'

# roles/example/tasks/redhat.yml
- yum:
    name: "httpd"
    state: present

# roles/example/tasks/debian.yml
- apt:
    name: "apache2"
    state: present

```


## Using Roles


## The original way of using Roles

The original way to use roles is via the roles: option for a given play:

```
---
- hosts: webservers
  roles:
     - common
     - webservers

```

This designates the following behaviors, for each role 'x':

- If roles/x/tasks/main.yml exists, tasks listed therein will be added to the play.

- If roles/x/handlers/main.yml exists, handlers listed therein will be added to the play.

- If roles/x/vars/main.yml exists, variables listed therein will be added to the play.

- If roles/x/defaults/main.yml exists, variables listed therein will be added to the play.

- If roles/x/meta/main.yml exists, any role dependencies listed therein will be added to the list of roles (1.3 and later).

- Any copy, script, template or include tasks (in the role) can reference files in roles/x/{files,templates,tasks}/ (dir depends on task) without having to path them relatively or absolutely.


When used in this manner, the order of execution for your playbook is as follows:

- Any `pre_tasks` defined in the play.

- Any `handlers` triggered so far will be run.

- Each role listed in `roles` will execute in turn. Any role dependencies defined in the roles `meta/main.yml` will be run first, subject to tag filtering and conditionals.

- Any `tasks` defined in the play.

- Any `handlers` triggered so far will be run.

- Any `post_tasks` defined in the play.

- Any `handlers` triggered so far will be run.


## The new way to use roles


You can now use roles inline with any other tasks using `import_role` or `include_role`:

```

---

- hosts: webservers
  tasks:
  - debug:
      msg: "before we run our role"
  - import_role:
      name: example
  - include_role:
      name: example
  - debug:
      msg: "after we ran our role"


```

1. The name used for the role can be a simple name, or it can be a fully qualified path:

```
---

- hosts: webservers
  roles:
    - role: '/path/to/my/roles/common'

```


2. Roles can accept other keywords:

```
---

- hosts: webservers
  roles:
    - common
    - role: foo_app_instance
      vars:
         dir: '/opt/a'
         app_port: 5000
    - role: foo_app_instance
      vars:
         dir: '/opt/b'


```

3. Using the newer syntax:


```

---

- hosts: webservers
  tasks:
  - include_role:
       name: foo_app_instance
    vars:
      dir: '/opt/a'
      app_port: 5000
  ...

```

4. You can conditionally import a role and execute it's tasks:

```

---

- hosts: webservers
  tasks:
  - include_role:
      name: some_role
    when: "ansible_facts['os_family'] == 'RedHat'"

```


5. You may wish to assign tags to the tasks inside the roles you specify. You can do:

```
---

- hosts: webservers
 roles:
   - role: bar
     tags: ["foo"]
   # using YAML shorthand, this is equivalent to the above

```

6. Using newer Syntax to assign tags

```

---

- hosts: webservers
  tasks:
  - import_role:
      name: foo
    tags:
    - bar
    - baz

```

7. On the other hand you might just want to tag the import of the role itself:

```

- hosts: webservers
  tasks:
  - include_role:
      name: bar
    tags:
     - foo


     ```

### Role Duplication and Execution

Ansible will only allow a role to execute once, even if defined multiple times, if the parameters defined on the role are not different for each definition. For example:

```

---
- hosts: webservers
  roles:
  - foo
  - foo

```

Given the above, the role foo will only be run once.


To make roles run more than once, there are **two** options:


Pass different parameters in each role definition.
Add allow_duplicates: true to the meta/main.yml file for the role.

**Example 1** - passing different parameters:

```
---
- hosts: webservers
  roles:
  - role: foo
    vars:
         message: "first"
  - { role: foo, vars: { message: "second" } }

  ```

In this example, because each role definition has different parameters, foo will run twice.


**Example 2** - using `allow_duplicates: true:`

```
# playbook.yml
---
- hosts: webservers
  roles:
  - foo
  - foo

# roles/foo/meta/main.yml
---
allow_duplicates: true

```

In this example, `foo` will run twice because we have explicitly enabled it to do so.

## Role Default Variables

Role default variables allow you to set default variables for included or dependent roles (see below). To create defaults, simply add a `defaults/main.yml` file in your role directory. These variables will have the lowest priority of any variables available, and can be easily overridden by any other variable, including inventory variables.

### Role Dependencies

Role dependencies allow you to automatically pull in other roles when using a role. Role dependencies are stored in the `meta/main.yml `file contained within the role directory, as noted above. This file should contain a list of roles and parameters to insert before the specified role


1. Example `roles/myapp/meta/main.yml`:

```
---
dependencies:
  - role: common
    vars:
      some_parameter: 3
  - role: apache
    vars:
      apache_port: 80
  - role: postgres
    vars:
      dbname: blarg
      other_parameter: 12
```

2. For example, a role named `car` depends on a role named `wheel` as follows:

```
---
dependencies:
- role: wheel
  vars:
     n: 1
- role: wheel
  vars:
     n: 2
- role: wheel
  vars:
     n: 3
- role: wheel
  vars:
     n: 4

```

3. The `wheel` role depends on two roles: `tire` and `brake`. The meta/main.yml for wheel would then contain the following:


```
---
dependencies:
- role: tire
- role: brake

```

4. And the `meta/main.yml` for `tire` and `brake` would contain the following:

```

---
allow_duplicates: true

```

The resulting order of execution would be:

```
tire(n=1)
brake(n=1)
wheel(n=1)
tire(n=2)
brake(n=2)
wheel(n=2)
...
car

```

Note that we did not have to use `allow_duplicates: true` for `wheel`, because each instance defined by `car` uses different parameter values.


# Role Search Path

Ansible will search for roles in the following way:

- A `roles/` directory, relative to the playbook file.

- By default, in `/etc/ansible/roles`


# Ansible Galaxy

Ansible Galaxy is a free site for finding, downloading, rating, and reviewing all kinds of community developed Ansible roles and can be a great way to get a jumpstart on your automation projects.

The client `ansible-galaxy` is included in Ansible. The Galaxy client allows you to download roles from Ansible Galaxy, and also provides an excellent default framework for creating your own roles.




## Variables

While automation exists to make it easier to make things repeatable, all systems are not exactly alike; some may require configuration that is slightly different from others. In some instances, the observed behavior or state of one system might influence how you configure other systems. For example, you might need to find out the IP address of a system and use it as a configuration value on another system.

Ansible uses variables to help deal with differences between systems.

To understand variables you'll also want to read [Conditionals](https://docs.ansible.com/ansible/latest/user_guide/playbooks_conditionals.html#playbooks-conditionals) and [Loops](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html#playbooks-loops).

For more information on variables: [Click Here](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html)



[< Back: Creating playbooks](https://github.com/sxcdennis/Ansible/blob/master/playbooks.md) || [Next: Examples of solving issues >](https://github.com/sxcdennis/Ansible/blob/master/issues.md)
