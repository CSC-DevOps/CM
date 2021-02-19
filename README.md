<!--
targets:
    - type: local
      name: local
    - type: bakerx
      name: config-server
-->

Configuration Management Workshop
----------------------------------

[Building a Configuration Server](README.md) | [Ansible Playbooks](Playbooks.md)

In this workshop, we'll cover the basics of setting up a simple configuration server, enabling ssh access, and using ansible to configure a simple web server.

![image](img/ansible-setup.png)


## Workshop

### Before you get started

Import this as a notebook or clone this repo locally. Ensure you [install latest version of docable](https://github.com/ottomatica/docable-notebooks/blob/master/docs/install.md), with multi-target support!

```bash
docable-server import https://github.com/CSC-DevOps/CM 
```

* Clone this repo with: `git clone https://github.com/CSC-DevOps/CM` 
* Make sure you have a shell open in the right directory: `cd CM`.
* Update opunit: `npm install opunit -g`. Ensure `opunit --version >= 0.4.4`.
* Update bakerx, take advantage of the new `--ip` option: `npm install ottomatica/bakerx -g` or `cd bakerx && npm install && npm update`. Ensure `bakerx --version` says `bakerx@0.6.7 virtcrud@1c9b4ba`.


### Checking progress on workshop

To check the configuration of the ansible server and web server, we will use `opunit` to run checks on the virtual machines listed in the inventory file. We can run checks from the top-level directory as follows: 

```bash | {type: 'command', target: 'local'}
opunit verify -i opunit_inventory.yml
```

## Creating your servers 

### The configuration server ⚒️

Let's create a configuration server. This server will be using a "push-based model", where we will be sending configuration commands to other external servers. We will install ansible.

Check virtualization.

```bash
$ VBoxManage list vms
```


Create the Virtual Machine.

```bash | {type: 'command', stream: true}
bakerx run config-server focal --ip 192.168.33.10 --sync
```

You should see bakerx create the virtual machine with a hostonly network.

```
Executing VBoxManage modifyvm "ansible-srv" --nic2 hostonly
Executing VBoxManage modifyvm "ansible-srv" --nictype2 virtio
```

Inside the ansible-srv, install ansible:

```bash | {type: 'command', target: 'config-server', stream: true}
sudo add-apt-repository ppa:ansible/ansible
sudo apt-get update
sudo apt-get install ansible -y
```


Verify that ansible was installed by running opunit.

```
$ opunit verify vagrant@192.168.33.10 --ssh_key ~/.bakerx/insecure_private_key -c test/ansible-srv.yml  
```

You should see the ansible check pass:

```bash
	version check
	    ✔   ansible --version: 2.9.4 > ^2.7.x => true 
```

### The web server 🌐

Let's create another virtual machine for the web server. 

```bash
$ bakerx run web-srv bionic --ip 192.168.33.100
```

You should see baker create the virtual machine.

## Creating a connection between your servers

You need a way to automatically connect to your server without having to manually authenicate each connection. We will create a pair of public/private keys for [authentication through ssh](https://www.ssh.com/ssh/public-key-authentication#sec-Asymmetric-Cryptography-Algorithms).

From your host machine, create a new public/private key pair, running the following command:

    ssh-keygen -t rsa -b 4096 -C "web-srv" -f web-srv -N ""

![ssh connection](img/connection.png)

After generating the keys, you need to copy them onto the servers. The private key (*secret*, tell know one 🤐), needs to be stored internally in the configuration server. The public key (🌐), needs to be _authorized_ for use on the web-server.

#### Storing the private key in the configuration server ⚒️

One nice trick is to use a copy utility to copy a file into your copy/paste buffer:

* Mac: `pbcopy < web-srv`
* Windows: `clip < web-srv`
* Linux: [Check out one of these options](https://superuser.com/a/288333/862331).

Let's go to the ansible-srv:

```
bakerx ssh ansible-srv
```

Using a file editor, paste and store your private key in a file:

```bash
ansible-srv $ vim ~/.ssh/web-srv
# Make sure key is not readable by others.
ansible-srv $ chmod 600 ~/.ssh/web-srv
# We're done here, go back to host
ansible-srv $ exit
```

Or use a simple command:

```bash
$ cat web-srv | ssh -i ~/.bakerx/insecure_private_key -o StrictHostKeyChecking=no vagrant@192.168.33.10 "cat > ~/.ssh/web-srv && chmod 600 ~/.ssh/web-srv"
```


#### Authorizing the public key on the web server 🌐

Copy the web-srv.pub file from your host.

Go inside the web-srv.

Using a file editor, add the public key to the list of authorized keys:

```bash
web-srv $ vim ~/.ssh/authorized_keys`
web-srv $ exit
```

Take care not to delete other entries, which may affect your ability to use bakerx to ssh into the machine. You also do not need to change the permissions of the file.

You may consider running this simple command to update the public key entry:

```bash
$ cat web-srv.pub | ssh -i ~/.bakerx/insecure_private_key -o StrictHostKeyChecking=no vagrant@192.168.33.100 "cat >> ~/.ssh/authorized_keys"
```

#### Testing your connection/Errors

Inside the ansible-srv, test your connection between the servers:

    ssh -i ~/.ssh/web-srv -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no vagrant@192.168.33.100

Note: You should also be able to make this same connection from your host machine, since you also have private key, locally.

If you see an error or prompt for a password, you have a problem with your key setup. Double check have pasted in the content in correctly (where you in insert mode in vim before pasting?). 

If you see this warning, you need to remember to perform the `chmod 600` in order to fix the permissions of the private key file:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0664 for '/home/vagrant/.ssh/web-srv' are too open.
```

## Performing commands over ssh

Once you have established a ssh connection between two servers, you have achieved an important milestone in configuration management.

You can now perform ad-hoc commands and even execute scripts on machines without having to manually log-in.

For example, when you run this command.

    ssh -i ~/.ssh/web-srv -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null vagrant@192.168.33.100 ls /

You can see the directory of the web-srv.

Practice: From your ansible-srv, create a file by executing the command below on the remote server. Verify it exists.

    touch ssh_test.txt

By this point, you should be able to pass the ssh setup checks in opunit: `opunit verify -i opunit_inventory.yml`.

## Ansible

While being able to run ad-hoc commands and scripts provides a useful capability, there are several constraints that make this impractical. 

Writing bash scripts can be error-prone. Most commands are not idempotent, meaning they may cause errors or unexpected behaviors if run multiple times on the same servers. Finally, configuration of servers is an inherently noisy problem, due to network issues and random service and hardware failures. This means, you often need to resume a configuration operation after experiencing partial failure.

Ansible is a tool for performing configuration changes on multiple machines. Ansible uses a push-based model for configuration management, performing idempotent commands over ssh, without requiring any agent running. The implementation is rather straightforward: _Ansible commands are translated into python snippets, and then copied over to the target machine, and executed. This requires that python is installed on the target machine_.

### Inventory

An inventory file allows ansible to define, group, and coordinate configuration management of multiple machines. At the most basic level, it basically lists the names of an asset and details about how to connect to it.

Inside the ansible-srv, edit the `inventory` file to include the ip address, user, and path to the private key:

```ini 
[web]
192.168.33.100 ansible_ssh_user=vagrant ansible_ssh_private_key_file=~/.ssh/web-srv 
[web:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

Now, run the ping test to verify ansible is able to talk to the web-srv!

    ansible all -m ping -i inventory

We should see a successful connection!

```
192.168.33.100 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Ansible commands

We can test our ability to use ansible for a simple configuration management task.

#### Installing nginx

Let's install a web server, called nginx (say like engine-X), on the web-srv VM. The web server will automatically start as a service.

    ansible all -b -m apt -i inventory -a 'pkg=nginx state=present update_cache=true'

Open a browser and enter in your node's ip address, e.g. http://192.168.33.100

#### Completing workshop checks

You should be able to verify all checks pass:

    opunit verify -i opunit_inventory.yml

Great work!

## Extra fun

Can you provision a remote VM (from digitalocean/etc.) and then add its details to the inventory? Can you run your nginx command on it too?

## Next steps

Run ansible commands can be useful for exploration and debugging. However, we want to be able to organize these commands into reusable configuration scripts.

Next workshop, we will learn about creating and running ansible playbooks.

## Common errors
Note, on older versions of ansible, you may fail to ping an item in your inventory if the server does not have python2. One workaround is to specify the python version explicitly. In newer versions of ansible, this will automatically be discovered by ansible. 

You can adjust the connection to account for this. Inside the inventory, add a new variable for all entries in `web` group:

```ini
[web:vars]
ansible_python_interpreter=/usr/bin/python3
```
