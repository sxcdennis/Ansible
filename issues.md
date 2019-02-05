# Example of using Ansible to solve an Issue

Now we know what Ansible is and how to use it. Let's try to use Ansible for what I believe it's trying to solve.



# Example 1

We're going to use Ansible as a tool to create and install a few things.

1. A webserver
2. A load balancer with several hosts


## Create Vagrant File & Bootstrap.sh


**Vagrantfile**

```

Vagrant.configure("2") do |config|

  # create mgmt node
  config.vm.define :mgmt do |mgmt_config|
      mgmt_config.vm.box = "ubuntu/trusty64"
      mgmt_config.vm.hostname = "mgmt"
      mgmt_config.vm.network :private_network, ip: "192.168.101.10"
      mgmt_config.vm.provider "virtualbox" do |vb|
        vb.memory = "2560"
      end
      mgmt_config.vm.provision :shell, path: "bootstrap-mgmt.sh"
  end

  # create load balancer
  config.vm.define :lb do |lb_config|
      lb_config.vm.box = "ubuntu/trusty64"
      lb_config.vm.hostname = "lb"
      lb_config.vm.network :private_network, ip: "192.168.101.11"
      lb_config.vm.network "forwarded_port", guest: 80, host: 8080
      lb_config.vm.provider "virtualbox" do |vb|
        vb.memory = "2560"
      end
  end

  # create some web servers
  (1..2).each do |i|
    config.vm.define "web#{i}" do |node|
        node.vm.box = "ubuntu/trusty64"
        node.vm.hostname = "web#{i}"
        node.vm.network :private_network, ip: "192.168.101.2#{i}"
        node.vm.network "forwarded_port", guest: 80, host: "808#{i}"
        node.vm.provider "virtualbox" do |vb|
          vb.memory = "2560"
        end
    end
  end

end

```

**boostrap.sh**

```

#!/usr/bin/env bash
# This shell script will configure the mgmt node

# install ansible
apt-get -y install software-properties-common
apt-add-repository -y ppa:ansible/ansible
apt-get update
apt-get -y install ansible

# configure hosts file for our internal network defined by Vagrantfile
cat >> /etc/hosts <<EOL
# vagrant environment nodes
192.168.10.10  mgmt
192.168.10.11  lb
192.168.10.21  web1
192.168.10.22  web2
192.168.10.23  web3
192.168.10.24  web4
EOL

```


### Log into  mgmt

`vagrant ssh mgmt`

### Edit Hosts file

**/etc/ansible/hosts**

```
[lb]
lb

[web]
web1
web2
web3
web4


```

## Configuaring ssh


1. **addsshkey.yml**

```
---
- hosts: all
  sudo: yes
  gather_facts: no
  remote_user: vagrant

  tasks:

  - name: install ssh key
    authorized_key: user=vagrant
                    key="{{ lookup('file', '/home/vagrant/.ssh/id_rsa.pub') }}"
                    state=present

```

2. Then use command

```

ssh-keyscan web1 web2 web3 web4 lb >> .ssh/known_hosts

```
3. Genereate public keygen

```

ssh-keygen -t rsa -b 2048

```

4. Then run playbook with --ask-pass with password vagrant

```

ansible-playbook addsshkey.yml --ask-pass

```

5. Verify trusts

```

ansible all -m ping

```


# Create playbook yml file


**example1.yml**


Hosts & Users, Tasks, Action Shorthand, and Handlers

```

---
# All
- hosts: all
  sudo: yes

  tasks:
  - name: install git
    apt:
      name: git
      state: installed

#web
- hosts: web
  sudo: yes

  tasks:
  - name: install nginx
    apt:
      name: nginx
      state: installed

  - name: configure our nginx.conf
    template: src=templates/nginx.conf.j2 dest=/etc/nginx/nginx.conf
    notify: restart nginx   

  - name: configure our /etc/nginx/site-available/default
    template: src=templates/nginx.default.j2 dest=/etc/nginx/sites-available/default
    notify: restart nginx

  - name: configure index.html for nginx
    template: src=templates/nginx.index.html.j2 dest=/usr/share/nginx/html/index.html

  handlers:
  - name: restart nginx
    service:
      name: nginx
      state: restarted
# lb
- hosts: lb
  sudo: yes

  tasks:
  - name: install haproxy and socat
    apt:  name={{item}} state=installed
    with_items:
      - haproxy
      - socat

  - name: enable haproxy
    lineinfile: dest=/etc/default/haproxy regexp="^ENABLED" line="ENABLED=1"
    notify: restart haproxy

  - name: deploy haproxy config
    template: src=templates/haproxy.cfg.j2 dest=/etc/haproxy/haproxy.cfg
    notify: restart haproxy

  handlers:

  - name: restart haproxy
    service:
      name: haproxy
      state: restarted

```

Now you may of noticed we have 4 templates. We will create them. Or you can view them under the **templates** directory on this repo.



**haproxy.cfg.j2**


```

# {{ ansible_managed }}
global
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        root
    group       root
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats level admin

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

    # enable stats uri
    stats enable
    stats uri /haproxy?stats

backend app
    {% for host in groups['lb'] %}
       listen mgmt {{ hostvars[host]['ansible_eth0']['ipv4']['address'] }}:80
    {% endfor %}
    balance     roundrobin
    {% for host in groups['web'] %}
        server {{ host }} {{ hostvars[host]['ansible_eth1']['ipv4']['address'] }} check port 80
    {% endfor %}

```



**nginx.conf.j2**


```

# {{ ansible_managed }}
user              www-data;

worker_processes  1;
pid        /var/run/nginx.pid;
worker_rlimit_nofile 1024;

events {
    worker_connections  512;
}


http {

        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        tcp_nopush "on";
        tcp_nodelay "on";
        #keepalive_timeout "65";
        access_log "/var/log/nginx/access.log";
        error_log "/var/log/nginx/error.log";
        server_tokens off;
        types_hash_max_size 2048;


        add_header X-Backend-Server $hostname;

        # disable cache used for testing
        add_header Cache-Control private;
        add_header Last-Modified "";
        sendfile off;
        expires off;
        etag off;

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}


```

**nginx.default.j2**


```

# {{ ansible_managed }}

server {

	listen 80;
	server_name {{ ansible_hostname }};
	root /usr/share/nginx/html;
	index index.html index.htm;

	location / {
		try_files $uri $uri/ =404;
	}

	error_page 404 /404.html;
	error_page 500 502 503 504 /50x.html;
	location = /50x.html {
		root /usr/share/nginx/html;
	}

}


```

**nginx.index.html.j2**


```

<html>
<title>Hello</title>
<h1> Hello1 </h1>
</html>


```

Now we run the ansible-playbook command.

```

ansible-playbook example1.yml

```


After that we check localhost:8080 on a browsers and see if our website works.


![ans3](https://github.com/sxcdennis/Ansible/blob/master/images/ans3.png?raw=true)

It works!

Now lets see if our load balancer works

```

http://localhost:8080/haproxy?stats

```

![ans4](https://github.com/sxcdennis/Ansible/blob/master/images/ans4.png?raw=true)



# Create roles

## Basic Overview


As stated before in the [roles](https://github.com/sxcdennis/Ansible/blob/master/roles.md) section, roles are a great way to organize your entire configuration. To create a role, you start by initializing into a directory. Followed by editing yml files and templates. Then run a yml file that calls the specific role. In this example, we'll take our previous example and turn it into a role.

To start we'll go over two commands:

- **ansible-galaxy init --offline**: Initiates files for ansible roles.

- **tree** Shows tree of ansible role directory.

Once you've initialized the environment, it'll create files and directories:

- `tasks` - contains the main list of tasks to be executed by the role.

- `handlers` - contains handlers, which may be used by this role or even anywhere outside this role.

- `defaults` - default variables for the role

- `vars` - other variables for the role

- `files` - contains files which can be deployed via this role.
templates - contains templates which can be deployed via this role.

- `meta` - defines some meta data for this role. See below for more details


## Creating the role

**1.** We'll start by using **ansible-galaxy init --offline**
and we'll make **3** roles: all, web, lb

```

mkdir -p /home/vagrant/roles/
ansible-galaxy init --offline /home/vagrant/roles/all
ansible-galaxy init --offline /home/vagrant/roles/web  
ansible-galaxy init --offline /home/vagrant/roles/lb


```

**2.** Use tree command to see successful files.

```

tree /home/vagrant/roles/all
tree /home/vagrant/roles/web
tree /home/vagrant/roles/lb

```

![ans5](https://github.com/sxcdennis/Ansible/blob/master/images/ans5.png?raw=true)


**3.** Convert **example1.yml** to split into each yml file on the roles of each tasks. We'll start with tasks for **all** .


**/home/vagrant/roles/all/tasks/main.yml**

```


---
# All
  - name: install git
    apt:
      name: git
      state: installed

```

**4a.**  We'll move to tasks for **web**

**/home/vagrant/roles/web/tasks/main.yml**

```

---
#TASKS
  - name: install nginx
    apt:
      name: nginx
      state: installed

  - name: configure our nginx.conf
    template: src=templates/nginx.conf.j2 dest=/etc/nginx/nginx.conf
    notify: restart nginx   

  - name: configure our /etc/nginx/site-available/default
    template: src=templates/nginx.default.j2 dest=/etc/nginx/sites-available/default
    notify: restart nginx

  - name: configure index.html for nginx
    template: src=templates/nginx.index.html.j2 dest=/usr/share/nginx/html/index.html

```

**4b.**  For **web** since we have a larger yml file we can actually split it up into different yml files and then import it.


**/home/vagrant/roles/web/tasks/main.yml**

```
---
# tasks for web

- import_tasks: /home/vagrant/roles/web/tasks/install.yml
- import_tasks: /home/vagrant/roles/web/tasks/configure.yml


```


**/home/vagrant/roles/web/tasks/install.yml**


```

# install nginx
---
- name: install nginx
  apt:
    name: nginx
    state: installed


```

**/home/vagrant/roles/web/tasks/configure.yml**


```

#configure web
---
- name: configure our nginx.conf
  template: src=/home/vagrant/roles/web/templates/nginx.conf.j2 dest=/etc/nginx/nginx.conf
  notify: restart nginx

- name: configure our /etc/nginx/site-available/default
  template: src=/home/vagrant/roles/web/templates/nginx.default.j2 dest=/etc/nginx/sites-available/default
  notify: restart nginx

- name: configure index.html for nginx
  template: src=/home/vagrant/roles/web/templates/nginx.index.html.j2 dest=/usr/share/nginx/html/index.html

```

copy templates over


```

cp /home/vagrant/templates* /home/vagrant/roles/web/templates

```

**5a.** Change main.yml for tasks

**/home/vagrant/roles/lb/tasks/main.yml**


```

---
- name: install haproxy and socat
  apt:  name={{item}} state=installed
  with_items:
    - haproxy
    - socat

- name: enable haproxy
  lineinfile: dest=/etc/default/haproxy regexp="^ENABLED" line="ENABLED=1"
  notify: restart haproxy

- name: deploy haproxy config
  template: src=/home/vagrant/roles/lb/templates/haproxy.cfg.j2 dest=/etc/haproxy/haproxy.cfg
  notify: restart haproxy

```


**5b.** Copy template over


```

cp template/haproxy.cfg.j2 /home/vagrant/roles/lb/templates/

```


**6.** Now we'll move onto handlers. We'll start with web handlers.

**/home/vagrant/roles/web/handlers/main.yml**

```
# web handlers
---
  - name: restart nginx
    service:
      name: nginx
      state: restarted
```

**7.** Then move onto handlers for lb

```
# lb handlers
---
  - name: restart haproxy
    service:
      name: haproxy
      state: restarted

```

**8.** Now we'll create run.yml for each one, starting with all.

**runall.yml**

```
---

- hosts: all
  roles:
  - all

```

**runweb.yml**


```
---

- hosts: web
  roles:
  - web

```

**runlb.yml**


```
---

- hosts: lb
  roles:
  - lb

```

**9.** We can create one file for all of these instead.

 **runeverything.yml**

 ```

 ---

- hosts: all
  roles:
  - all

- hosts: web
  roles:
  - web

- hosts: lb
  roles:
  - lb

```

**10.** Now we can run the file


```

ansible-playbook runeverything.yml

```


![ans7](https://github.com/sxcdennis/Ansible/blob/master/images/ans7.png?raw=true)

Hooray! It works!

We can even check the website and haproxy locally.

![ans8](https://github.com/sxcdennis/Ansible/blob/master/images/ans8.png?raw=true)


![ans9](https://github.com/sxcdennis/Ansible/blob/master/images/ans9.png?raw=true)


## Summary

As you can see Ansible can be quite useful. This is just the tip of it. As roles and templating can get quite complex in bigger environments.


[< Back: Reusable Playbooks: Includes, Imports, and Roles](https://github.com/sxcdennis/Ansible/blob/master/roles.md)
