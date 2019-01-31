# Basic Ansible Overview

# Hosts Inventory
Host inventory file is a INI file that is basically a listing of hosts that you want to manage with Ansible, you can group them together under random headings too.

**Example**

```

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

**Example:**
