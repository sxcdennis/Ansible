# Using Vagrant and Ansible

Vagrant is a good tool to manage virtual machine environments.


**Note:** Most of this information was taken out of the Ansible Documentation and put together to be read a bit easier.

## Installing Vagrant

Find the download link to the rpm at
https://www.vagrantup.com/downloads.html

Then use that link with yum or apt command.

```

yum -y install https://releases.hashicorp.com/vagrant/2.2.3/vagrant_2.2.3_x86_64.rpm

```


## Vagrant Setup

The first step once you've installed Vagrant is to create a **Vagrantfile** and customize it to suit your needs. Here is a quick example that includes a section to use the Ansible provisioner to manage a single machine:


```

Vagrant.require_version ">= 1.7.0"

Vagrant.configure(2) do |config|

  config.vm.box = "ubuntu/trusty64"
  config.ssh.insert_key = false

  config.vm.provision "ansible" do |ansible|
    ansible.verbose = "v"
    ansible.playbook = "playbook.yml"
  end
end

```

Notice the **config.vm.provision** section that refers to an Ansible playbook called **playbook.yml** in the same directory as the Vagrantfile. Vagrant runs the provisioner once the virtual machine has booted and is ready for SSH access.

**ansible.verbose** shows what **ansbile.playbook** is using.



To re-run a playbook on an exisitng VM use

```

vagrant provision

```
This will re-read the playbook.yml file or whatever is configured in your Vagrant file.

### Running Ansible Manually on Vagrant

Sometimes you might want to run Ansible manually on the virutal machines. This option is faster than using **vagrant provision**
With our Vagrantfile example, Vagrant automatically creates an Ansible inventory file in **.vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory**. This inventory is configured according to the SSH tunnel that Vagrant automatically creates. A typical automatically-created inventory file for a single machine environment may look something like this:


```

default ansible_ssh_host=127.0.0.1 ansible_ssh_port=2222

```


If you want to run Ansible manually, you will want to make sure to pass ansible or ansible-playbook commands the correct arguments, at least for the username, the SSH private key and the inventory.

Here is an example using the Vagrant global insecure key (config.ssh.insert_key must be set to false in your Vagrantfile):

```

$ ansible-playbook --private-key=~/.vagrant.d/insecure_private_key -u vagrant -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory playbook.yml

```
Here is a second example using the random private key that Vagrant automatically configures for each new VM (each key is stored in a path like .vagrant/machines/[machine name]/[provider]/private_key):

```

$ ansible-playbook --private-key=.vagrant/machines/default/virtualbox/private_key -u vagrant -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory playbook.yml

```


[< Back: Basic YMAL Syntax](https://github.com/sxcdennis/Ansible/blob/master/ymal.md) || [Next: Creating inventory files >](https://github.com/sxcdennis/Ansible/blob/master/inventory.md)
