#  Inventory Files

Typically the **/etc/ansible/ansible.cfg** file will show the behavior of checking for hosts files. For example inside the file might look like this:

```

[defaults]
hostfile = /vagrant/inventory.ini

```

Which will point your default host file to **/vagrant/inventory.ini**

If you are dealing with **Dynamic Inventories** (Clobber, AWS EC2, OpenStack etc..), you can use this guide on the Ansible website: [Click Here](https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html#intro-dynamic-inventory)


Host inventory file is a INI file that is basically a listing of hosts that you want to manage with Ansible, you can group them together under random headings too.

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



If you are adding a lot of hosts following similar patterns, you can do this rather than listing each hostname:

```
[webservers]
www[01:50].example.com

```
For numeric patterns, leading zeros can be included or removed, as desired. Ranges are inclusive. You can also define alphabetic ranges:

```
[databases]
db-[a:f].example.com

```
You can also select the connection type and user on a per host basis:

```
[targets]

localhost              ansible_connection=local
other1.example.com     ansible_connection=ssh        ansible_user=mpdehaan
other2.example.com     ansible_connection=ssh        ansible_user=mdehaan

```




 [< Back: Installing Ansible Automatically with Vagrant Boxes](https://github.com/sxcdennis/Ansible/blob/master/vagrant.md) || [Next: Creating playbooks >](https://github.com/sxcdennis/Ansible/blob/master/playbooks.md)
