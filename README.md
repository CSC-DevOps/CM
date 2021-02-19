
Configuration Management Workshop
----------------------------------

> In this workshop, we'll cover the basics of setting up a simple configuration server, enabling ssh access, and using ansible to configure a simple web server. 

Part 1. [Configuration Management Workshop](README.md)  ⬅️   
Part 2. [Building a Configuration Server](CM.md)   
Part 3. [Ansible Playbooks](Playbooks.md)  

## Workshop

An overview of the components we will set up can be seen here:

![image](img/ansible-setup.png)

### Before you get started

Import this as a notebook or clone this repo locally. Ensure you [install latest version of docable](https://github.com/ottomatica/docable-notebooks/blob/master/docs/install.md), with multi-target support!

```bash
docable-server import https://github.com/CSC-DevOps/CM 
```

* Clone this repo with: `git clone https://github.com/CSC-DevOps/CM` 
* Make sure you have a shell open in the right directory: `cd CM`.
* Update opunit: `npm install opunit -g`. Ensure `opunit --version >= 0.4.4`.
* Update bakerx, take advantage of the new `--ip` option: `npm install ottomatica/bakerx -g` or `cd bakerx && npm install && npm update`. Ensure `bakerx --version` says `bakerx@0.6.7 virtcrud@1c9b4ba`.

