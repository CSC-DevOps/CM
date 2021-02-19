
Configuration Management Workshop
----------------------------------

> In this workshop, we'll cover the basics of setting up a simple configuration server, enabling ssh access, and using ansible to configure a simple web server. 

Part 1. [Configuration Management Workshop](README.md)  ⬅️   
Part 2. [Building a Configuration Server](CM.md)   
Part 3. [Ansible Playbooks](Playbooks.md)  

## Before you get started

Import this as a notebook or clone this repo locally.  

Also, ensure you [install latest version of docable](https://github.com/ottomatica/docable-notebooks/blob/master/docs/install.md), with multi-target support!

```bash
docable-server import https://github.com/CSC-DevOps/CM 
```

### Workshop setup

We will be using two VMs, one will be called the 🎛️  `config-server`, and the other called the 🌍  `web-srv`.

1. Make sure you already have a focal image, pulled locally.

```bash | {type: 'command'}
bakerx pull focal cloud-images.ubuntu.com
```

#### Config-server VM

2. Create the Virtual Machine for the 🎛️  `config-server`.

```bash | {type: 'command', stream: true, failed_when: "exitCode != 0"}
bakerx run config-server focal --ip 192.168.33.10 --sync
```

📝  There are two interesting things to note about this VM.
* 1) You should see a second NIC created with a host-only network and assigned a static ip address.
* 2) Because the `--sync` option was provided, the VM will also mount your current directory on your host computer, which is accessible via `/bakerx` (like a volume in docker).

#### Web-server VM

3. Create the Virtual Machine for the 🌍  `web-srv`.

```bash | {type: 'command', stream: true, failed_when: "exitCode != 0"}
bakerx run web-srv focal --ip 192.168.33.100
```

#### Verifying setup is correct

4. Using the terminal, run a couple of checks to make sure everything is okay. 

```bash | {type: 'terminal'}
```

First, from our host computer, let's make sure we can run `ping 192.168.33.10` and `ping 192.168.33.100`. 
If this works, this should mean both our VM's correctly have host-only networking setup, and they both have an ip address you can use to connect to them.

Second, run `bakerx ssh config-server` to access your 🎛️  `config-server`. Then make sure your VM has properly mounted your host's computer's directory.

You should see something like:

```bash
vagrant@ubuntu-focal:~$ ls /bakerx
CM.md  Playbooks.md  README.md  examples  img  opunit_inventory.yml  test  yaml
```