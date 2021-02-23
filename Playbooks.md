<!--
targets:
    - type: local
      name: local
    - type: bakerx
      name: config-server
-->

Part 1. [Configuration Management Workshop](README.md)  
Part 2. [Building a Configuration Server](CM.md)  
Part 3. [Ansible Playbooks](Playbooks.md)  ⬅️   

Although you can run ad-hoc commands in ansible, in practice, you'll largely be expected to create ansible playbooks. Ansible playbooks are essentially files formatted as [yaml](http://docs.ansible.com/ansible/YAMLSyntax.html).

## Hello Playbooks

Let's confirm our ansible setup still works. If not, you can double back to Part 2 and make sure you have everything working first.

```bash | {type: 'command', target: 'config-server'}
ansible all -i inventory -m ping
```

What if we didn't want to have to run manual commands? 

An *ansible playbook* allows you to execute a collection of configuration management tasks from a file. Here is an example of the same ansible command, but expressed as an ansible playbook:

```yaml
---
- hosts: all
  gather_facts: no
  tasks:
  - name: ping all hosts
    ping:
```

You can run this playbook by running:

```bash | {type: 'command', target: 'config-server'}
ansible-playbook /bakerx/examples/ping.yml -i inventory
```

### Understanding yaml

Understanding and writing ansible scripts is largely a function of understanding _YAML_ (YAML Ain't Markup Language or formally Yet Another Markup Language), which can be thought as [a superset of JSON](https://stackoverflow.com/questions/1726802/what-is-the-difference-between-yaml-and-json/1729545#1729545).

You can read a [nice overview](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html) on the syntax. But first, we will understand it by parsing it. *Note*: YAML can be very picky about indentation.

**ACTIVITY**: Print name and command property of each object in "tasks" list.

```js | {type: 'script', target: 'local'}
const yaml = require('js-yaml');
let yamlFile = `
top:
  children:
    - one
    - two
    - three
  tasks:
    - name: Install nodejs
      apt: pkg='nodejs' state='present'
    - name: Run ls command
      command: ls
`;
var doc = yaml.load(yamlFile, 'utf8');
```

For more practice, you can try this [interactive notebooks for parsing yaml](https://docable.cloud/chrisparnin/notebooks/nodejs/yaml/yaml-practice.md).


### Basic playbook structure

Let's break down our example ping playbook to understand it's basic parts.

```yaml
---
- hosts: all
  gather_facts: no
  tasks:
  - name: ping all hosts
    ping:
```

##### Hosts 

```yaml | {type:'info', range: {start:2, end: 2}}
--- # Headline denoting start of yaml document.
# This the first object in the list of playbooks. You can have more than one.
- hosts: all # Declares hosts="all" for the playbook object.
```

Hosts refers to entries in your inventory. Consider the following inventory:

```ini
mail.example.com

[webservers]
www1.example.com
www2.example.com

[dbservers]
db1.example.com
db2.example.com
db3.example.com
```

In this case, `hosts: all` would every server in the inventory (all six). If we set `hosts: webservers`, then we would only refer to `www1.example.com` and `www2.example.com`. Finally, it is possible to run commands on the local server with ansible itself, by providing `hosts: localhost`.

##### Gather facts

```yaml
  gather_facts: no
```

When ansible connects to a server over ssh, it will run special python modules for gather statistics about the server and obtaining state needed for running the playbook. **It can be desirable to turn this off if** we're establishing basic connectivity for two reasons: 
   1. for performance, 
   2. for bootstraping a server or when repairs must be done against server. Imagine python was broken or missing on your server, ansible would not be able to work. However, by turning off `gather_facts`, and then running `- raw: sudo apt-get install -y python-minimal`, you could fix this problem.

##### Tasks

Tasks are the essential component of a playbook. Here is where you run actual commands on the server.

```yaml | {type: 'info', range: {start: 1, end: 2}}
  tasks:
    - name: ping all hosts # purely for documentation, this is what is printed out when you run the playbook.
      ping: # We are running the ping module. Notice that no arguments are needed.
```

##### Inline style versus explicit style

You might often see playbooks written as follows:

```yaml
  tasks:
    - name: Install nodejs
      apt: pkg='nodejs' state='present' # We are telling the apt module to essentially run `apt-get install nodejs`.
```

Notice that we could write our apt install task in a different way.

```yaml
  tasks:
    - name: Install nodejs
      apt: 
        pkg: nodejs
        state: present
```

Both snippets will do absolutely the same thing. With the "inline style" ansible will parse out the object properties from the string it is given. In the second snippet, we explicitly specify the `pkg` and `state` properties for the apt module.

## Playbook concepts

Playbooks require several essential concepts, such as modules, variables, templates, roles, and more.

### Modules

Ansible's value is in the rich library of modules it provides to make it easier to run commands on a server.

* Install packages ([apt](https://docs.ansible.com/ansible/2.4/apt_module.html), [yum](https://docs.ansible.com/ansible/2.4/yum_module.html), [dnf](https://docs.ansible.com/ansible/2.4/dnf_module.html)).

```yaml | {type: 'info', range: {start:1,end:1}}
- name: Install latest version of "openjdk-6-jdk" ignoring "install-recommends"
  apt:
    name: openjdk-6-jdk
    state: latest
    install_recommends: no
```


* Manage services on the server.

```yaml | {type: 'info', range: {start:2,end:3}}
- name: Start service httpd, if not started
  service:
    name: httpd
    state: started
```

* Perform file operations ([file](https://docs.ansible.com/ansible/2.4/file_module.html), [copy](https://docs.ansible.com/ansible/2.4/copy_module.html), [template](https://docs.ansible.com/ansible/2.4/template_module.html)):

```yaml
- name: Copy file with owner and permission, using symbolic representation
  copy:
    src: /srv/myfiles/foo.conf
    dest: /etc/foo.conf
    owner: foo
    group: foo
    mode: u=rw,g=r,o=r
```

### Variables

Ansible provides [variables](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html) as a way to avoid hard-coding configuration values inside ansible tasks.

You can declare variables as part of the top-level playbook object and reference variables inside of the playbook by using `"{{variable_name}}"`.

```yaml | {type: 'info', range: {start: 9, end: 11}}
  vars:
    user: "mattermost"
    group: "mattermost"
    file: "config.conf"
      
  tasks:
    - name: Copy file with owner and permission, using symbolic representation
      copy:
        src: /srv/myfiles/foo.conf
        dest: "/etc/{{file}}"
        owner: "{{user}}"
        group: "{{group}}"
        mode: u=rw,g=r,o=r
```

You can also load variables from an external file with `vars_files:`.

```yaml
  vars_files:
    - /vars/external_vars.yml
```

Variables can also be loaded as prompts, with `vars_prompt`, or passed on the command line: `--extra-vars "version=1.23.45 other_variable=foo"`.

### Loops

Ansible provides the ability to [loop over lists](https://docs.ansible.com/ansible/2.4/playbooks_loops.html) defined inside a variable. To use this capability, you simply define a `with_items` property on a task object.

```yaml
- hosts: all

  vars: 
    packages:
      - gcc
      - debconf-utils
      - git

  tasks: 
    - name: Install git
      apt: pkg={{item}} state=present update_cache=yes
      become: yes
      with_items: "{{packages}}"
```

### Templates

Templates are powerful ways to setup basic configuration settings without hard coding values.
When you use a template, it will get the template, fill in any parameters, and then copy the file over to the destination. 

For example, a template file, called `.my.cnf.j2` containing the following:

```ini
[client]
user=root
password={{root_db_password}}
local_infile=1
```

Templates can be instantiated and copied to a server with the following task.

```yaml
- name: copy .my.cnf file with mysql root password credentials
  template: src=templates/root/.my.cnf dest={{ ansible_env.HOME}}/.my.cnf owner={{mysql_account}} mode=0600
```

This can be useful for setting up complex configuration files such as apache, mysql, or jenkins. 

### Vaults

It is possible to store secrets encrypted using [ansible-vault](http://docs.ansible.com/ansible/playbooks_vault.html). This is recommended if you need to store tokens, passwords, or ssh keys.

## Practicing commands

The simpliest way to get started is to try executing some basic tasks inside of a playbook.
In examples folder, execute the [commands.yml](examples/commands.yml) playbook.

```bash | {type: 'command', target: 'config-server'}
ansible-playbook /bakerx/examples/commands.yml -i inventory
```

This will ensure a .ssh directory exists and creates a ssh key. Inspect the directory and ensure it exists. Notice that this runs on your ansible server (`hosts: localhost`).

**ACTIVITY:**

* Run the command again. You should see changes=0.
* Manually delete the ssk keypair (`demo`, `demo.pub`) that was generated. Run the command again.

```bash | {type: 'terminal', target: 'config-server', 'background-color': '#C80815'}
```

### Practice updating script

Given this yaml fragment, attempt to write a playbook called node.yml that installs node.js on servers in your inventory. 

**Hint**: You will need to a) add a hosts section, b) Ensure apt-get update has been run at least once with `update_cache:yes`, c) ensure you can run with privileges (`become: yes`).

```yaml | {type: 'file', path: '/bakerx/examples/node.yaml', target: 'config-server'}
  tasks:
    - name: Install nodejs
      apt: 
        pkg: nodejs
        state: present
```

Once you've created your playbook, try completing the command to run it:

```bash | {type: 'command', target: 'config-server'}
ansible-playbook 
```

### Output/Register/Conditions

When a module is executed, you can save the results of that command using `register`.
Further, you can use conditions to control whether a task is executed with `when`.
You can control whether a command is considered changed, using `changed_when`.

For example, this command will save the output of `echo` into the variable `out`.
Its changed status will change depending on whether there is a list item with z.

```yaml
- action: command echo {{item}}
  register: out
  changed_when: "'z' in out.stdout"
  with_items:
    - hello
    - foo
    - bye
```

Mixing `register`, `changed_when`, and `with_items` can get tricky. Based on context, sometimes the variable saved in register will change per item during task execution. Othertimes, when all done contain a list of all items and their results. For some more details:
[see this example](http://stackoverflow.com/a/41292571/547112).

```bash | {type: 'terminal', target: 'config-server', 'background-color': '#7e050d'}
```

### Enabling Idempotence for commands

By default, most ansible modules are written to be idempotent. However, if you need to use the `command` or `shell` module, then the operations performed will not likely be idempotent.

**ACTIVITY**: The following playbook downloads a file using `wget`. If you run this playbook twice, the shell module doesn't have a way to check if the task has already been done. It isn't idempotent and it will wastefully download the file over and over again!

```yaml
---
- hosts: all

  vars: 
    - exchange_item: 3dprinting.meta.stackexchange.com.7z
    - url: "http://archive.org/download/stackexchange/{{exchange_item}}"
    - dest_dir: /tmp

  tasks:
    - name: Get stats of a file
      ansible.builtin.stat:
        path: {{dest_dir}}/{{exchange_item}}
        register: st
    - name: get stack overflow data
      shell: "wget -nv -P {{dest_dir}} {{url}}"
```

* Edit the file in `/bakerx/examples/results.yml` so that there is a task that [first checks if the file already exists](https://docs.ansible.com/ansible/devel/collections/ansible/builtin/stat_module.html) in /tmp. Only download when it doesn't exist by adding a `when` condition. *Note*: that `creates` option to shell is an alternative option, however, it there are some limits. Example: `creates={{dest_dir}}/{{exchange_item}}"`.

## Content Layout/Roles

Finally, as your playbooks grow more complex in size, you will start to think about ways to organize and seperate out tasks.

Ansible describes some recommended way to organize your playbooks:
http://docs.ansible.com/ansible/playbooks_best_practices.html#content-organization

An important part of organization is [roles](http://docs.ansible.com/ansible/playbooks_roles.html). Roles allow for you to essentially "include" in other playbooks. However, they use a particular layout to organize content.

Folder layout:
```
- roles/
  - setup/
    - tasks/
      - main.yml
    - templates/
      .my.cnf
- playbook.yml
```

An example use of roles:

```yaml
---
- hosts: dbservers
  vars_prompt:
    - name: run_schema_import
      prompt: "Do import of data (schema/import)? Y: Will destory existing data. N: Will skip"
      default: "N"
  # This loads sudo password from encrypted vault
  vars_files:
    - vault/dbservers.vars
  vars: 
    - VersionBotDB: VersionsDB

  roles: 
    - { role: schema, when: run_schema_import == "Y" or run_schema_import == "y" }
    - indexes
    - import
    - users
```

### Everything else

The ansible community supports a wide collection of modules to do amazing things. It is possible to build custom modules or programmatically interface with ansible. Learning ansible will be tricky, but the investiment can be worth it!
