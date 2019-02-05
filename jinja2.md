# Jinja2

## A Basic Example

In the following task, I am using the template module on the example1.j2 file which will replace the default variables with values given in the playbook.

**Playbook.yml**


```

---
- hosts: all
  vars:
    variable1: 'Hello...!!!'
    variable2: 'My first playbook using template'
  tasks:
    - name: Basic Template Example
      template:
        src: example1.j2
        dest: /home/Ansible/output.txt


```


**example1.j2**

```
{{ variable1 }}
No effects on this line
{{ variable2 }}

```

**output.txt**

```
Hello...!!!
No effects on this line
My first playbook using template

```

As you can see, both variables in the example1.j2 are replaced by their values. At the bare minimum, you need to have two parameters when using the Ansible template module.

- **src:** The source of the template file. This can be a relative or absolute path.

- **dest:** The destination path on the remote server


## Additional attributes of the template module

These are some of the other parameters which we can use to change some default behavior of template module :

- **force** – If the destination file already exists, then this parameter decides whether it should be replaced or not. By default, the value is ‘yes’.

- **mode** – If you want to set the permissions for the destination file explicitly, then you can use this parameter.

- **backup** – If you want a backup file to be created in the destination directory, you should set the value of the backup parameter to ‘yes’. By default, the value is ‘no’. The backup file will be created every time there is a change in the destination directory.

- **group** – Name of the group that should own the file/directory. It is similar to executing chown for a file in Linux systems.


## Using lists in Ansible templates


In the next example, I’ll be using the template module to print all the items present in a list using the for the loop.

**Playbook.yml**

```

- hosts: all
  vars:
    list1: ['Apple','Banana','Cat', 'Dog']
  tasks:
    - name: Template Loop example.
    - template:
        src: example2.j2
        dest: /home/Ansible/output.txt

```

**example2.j2**

```
Example of template module loop with a list.
{% for item in list1 %}
  {{ item }}
{% endfor %}

```

**output.txt**

```
Example of template module loop with a list.
Apple
Banana
Cat
Dog

```


We can see that after each iteration, a newline is added. The more practical approach of for loops is when we have to add a specific port to a large number of hosts and it is too tedious to manually update the inventory file. We can simply write the following code to update the same

```

host.list={% for node in groups['nodes'] %}{{ node }}:5672{% endfor %}

```


## Working With Multiple Files in Ansible



We can use the with_items parameter on a dictionary to render multiple files. For example, if we want to render three templates each with different source and destination, with_items parameter can be put to use.  

```
- hosts: all
  tasks:
    - name: Template with_items example.
      template:
        src: "{{ item.src }}"
        dest: "{{ item.dest }}"
      with_items:
        - {src: 'example.j2',dest: '/home/Ansible/output.txt'}
        - {src: 'example1.j2',dest: '/home/Ansible/output1.txt'}
        - {src: 'example2.j2',dest: '/home/Ansible/output2.txt'}

```


## Working With Filters

Filters are a way of transforming template expressions from one kind of data into another. They take some arguments, process them, and return the result. There are a lot of [builtin filters](http://jinja.pocoo.org/docs/2.10/templates/#builtin-filters) which can be used in the Ansible playbook. For example, in the following code myvar is a variable; Ansible will pass myvar to the Jinja2 filter as an argument. The Jinja2 filter will then process it and return the resulting data.

```
    {{ myvar | filter }}
```


Templating happens on the Ansible controller, not on the task’s target host, so filters also execute on the controller as they manipulate local data. Some common uses of filters are as follows :

**formatting data** – These filters will take a data structure in a template and render it in a slightly different format. These are occasionally useful for debugging. It may be reading in some already formatted data.

```
{{ some_variable | to_json }}
{{ some_variable | from_json }}.

```

**IP address filter** – Used to test if a string is a valid IP address. IP address filter can also be used to extract specific information from an IP address.

```
{{ myvar | ipaddr }}.

```

**URL Split Filter** – The urlsplit filter extracts the fragment, hostname, netloc, password, path, port, query, scheme, and username from an URL. With no arguments, returns a dictionary of all the fields.

```
{{ "http://user:password@www.abc.com:9000/dir/index.html?query=term#frament" | urlsplit('hostname') }}

```

**Regular Expression Filters** – Used to search a string with a regex, use the "regex_search" filter.

```

{{ 'foobar' | regex_search('(foo)') }}

```


**List Filters** – These filters all operate on list variables.  

```

{{ list1 | min }}
{{ list1 | unique }}
{{ list1 | union(list2) }}.

```


Defining default values for undefined variables: Jinja2 provides a useful ‘default’ filter, that is often a better approach to failing if a variable is not defined.

```{{ some_variable | default(5) }}```
