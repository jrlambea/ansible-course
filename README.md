# Ansible Course Notes

Oficial docs: https://docs.ansible.com/ansible/latest/index.html

## Install

Debian:

```bash
echo "deb http://ppa.launchpad.net/ansible/ansible/ubuntu trusty main" >> /etc/apt/sources.list && \
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 93C4A3FD7BB9C367 && \
sudo apt update && \
sudo apt install ansible
```

RHEL 8:

```bash
sudo subscription-manager repos --enable ansible-2.9-for-rhel-8-x86_64-rpms
sudo yum install ansible
```

## Ansible Inventory

Ansible use `ssh` for Linux and `powershell remoting` for Linux to connect with the machines of the inventory. It is agentless.

Ansible uses inventory files, if no inventory file is specified, Ansible uses by default `/etc/ansible/hosts`.

example:

```text
server1.company.com
server2.company.com

[web]
server3.company.com
server4.company.com
```

It is possible to make aliases for servers:

```text
web ansible_host=server1.company.com ansible_connection=ssh   ansible_user=root
db  ansible_host=server2.company.com ansible_connection=winrm ansible_user=admin
db  ansible_host=server2.company.com ansible_connection=winrm ansible_ssh_pass=P@ssword
localhost ansible_connection=localhost
```

There is more inventory parameters:
```text
ansible_connection
ansible_port
ansible_user
ansible_ssh_pass
```

Check target server:

```bash
ansible server1.company.com -m ping -i inventory.txt
```

**Not recommendend.** In order to prevent errors while connecting with servers that we doesn't have the SSH key fingerprint, edit `ansible.cfg` and add or uncomment:

```ini
host_key_checking = False
```

Is also **not recommended** to use plain passwords for connect with the target servers, it's better to use SSH keys.

Other uses:

```bash
ansible localhost -a "/sbin/reboot"
```

## Ansible Playbook

YAML format files. Dictionaries with `name`, `hosts` and `tasks` entries. Example:

```yaml
- name: Play1
  hosts: localhost
  tasks:
  - name: Execute command 'date'
    command: date

  - name: Execute script on server
    script: test_script.sh

  - name: Install httpd service
    yum:
      name: httpd
      state: present

  - name: Start web server
    service:
      name: httpd
      state: started
```

`command`, `script`, `yum` and `service` are modules. There are hundreds of modules out of the box, for more info:

```bash
ansible-doc -l
```

To run playbooks:

```bash
ansible-playbook playbook.yml
```

or

```bash
ansible-playbook playbook.yml -i inventory.txt
```

Aditionals parameters:

```bash
ansible-playbook --help
```

Example of simple playbook with `ping` module:

```yaml
- name: ping host
  hosts: localhost
  tasks:
  - name: ping the host
    ping:
```

## Ansible Modules

There is a lot of modules available, e.g: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html

The parameters of modules can be written in one line, or as yaml list.

One line:

```yaml
- name: One line module example playbook
  hosts: all
  tasks:
  - name: This is a one line user
    user: name=john uid=1400 group=development
```

Multiline:

```yaml
- name: One line module example playbook
  hosts: all
  tasks:
  - name: This is a one line user
    user:
      name: john
      uid: 1400
      group: development
```

## Ansible Variables

The variables can be defined in a file apart, as key-value:

```yaml
variable1: value1
variable2: value2
```

Or within the Playbook YAML, with the dictionary `vars`, in order to use the variable, we have to do Jinja2 Templating, use the "{{}}" to surround the name of the variable to use:

```yaml
- name: One line module example playbook
  hosts: all
  vars:
    uid: 1400
  tasks:
  - name: This is a one line user
    user:
      name: john
      uid: '{{uid}}'
      group: development
```

Also we can hardcode the variables and values in the inventory file:

```text
Web http_port=8081 snmp_port=161-162 inter_ip_range=192.0.2.0
```

## Conditionals

Conditionals statements can be used, e.g.:

### When

```yaml
- name: Install NGINX
  hosts: all
  tasks:
  - name: Install NGINX on Debian
    apt:
      name: nginx
      state: present
    when: ansible_os_family == "Debian"
  - name: Install NGINX on Redhat
    yum:
      name: nginx
      state: present
  when: ansible_os_family == "RedHat" or
        ansible_os_family == "SUSE"
```   

### Conditionals in Loops

```yaml
- name: Install Softwares
  hosts: all
  vars:
    packages:
    - name: nginx
      required: True
    - name: mysql
      required: True
    - name: apache
      required: False
  tasks:
  - name: Install "{{ item.name }}" on Debian
    apt:
      name: "{{item.name}}"
      state: present
    when: item.required == True
    loop: "{{packages}}"
```   

### Conditionals & Register

```yaml
- name: Check status of a service and email if its down
  hosts: localhost
  tasks:
    - command: service httpd status
      register: result
    - mail:
      to: admin@company.com
      subject: Service Alert
      body: Httpd Service is down
      when: result.stdout.find('down') != -1
```   

## Loops

```yaml
- name: Create users
  hosts: localhost
  tasks:
    - user: name='{{item}}' state=present
      loop:
      - joe
      - george
      - ravi
      - mani
```

```yaml
- name: Create users
  hosts: localhost
  tasks:
    - user: name='{{item.name}}' state={{item.uid}}
      loop:
      - name: joe
        uid: 1010
      - name: george
        uid: 1011
      - name: ravi
        uid: 1012
      - name: mani
        uid: 1013
```

With `with_*`:

```yaml
name: Create users
hosts: localhost
tasks:
- user: name='{{item}}' state=present
  with_items:
    - joe
    - george
    - ravi
    - mani
```

```yaml
- name: 'Print list of fruits'
  hosts: localhost
  vars:
    fruits:
    - Apple
    - Banana
    - Grapes
    - Orange
  tasks:
  - command: 'echo "{{item}}"'
    with_items: "{{fruits}}"
```

All that appears with the prefix `with_` depends on `lookup plugins`. To know which `lookup plugins` are available run `ansible-doc -t lookup -l`:

## Ansible Roles

Role is a grouping of tasks, that can be assigned to a playbook, e.g.:

```yaml
# MySQL Role
tasks:
  - name: Install PreRequisites
    yum: name=pre-req-packages state=present

  - name: Install MySQL Packages
    yum: name=mysql state=present

  - name: Start MySQL Service
    service: name=mysql state=started

  - name: Configure Database
    mysql_db: name=db1 state=present
```

Use in a Playbook:

```yaml
- name: Install and Configure MySQL
  hosts: db-server
  roles:
    - mysql
```

Or as dictionary (`become` is to escalate):

```yaml
- name: Install and Configure MySQL
  hosts: db-server
  roles:
    - role: mysql
      become: yes
```

It's needed to follow a concrete directory architecture, this can create automatically with `ansible-galaxy init {{name-of-role}}`

```text
.
├── defaults
│   └── main.yml
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── README.md
├── tasks
│   └── main.yml
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

This new folder should be placed inside a `roles` folder on the same folder that the Playbook resides, or place this new folder into `/etc/ansible/roles` (or wherever is configured in the setting `roles_path` inside the configuration file `ansible.cfg`).

Differents roles can be finded through the online searcher `ansible-galaxy search {{name-of-role}}`.

And installed by `ansible-galaxy install geerlingguy.mysql`.

List roles installed with `ansible-galaxy list`

## Advanced Topics

- Ansible Control Machine can only be Linux and not Windows
- Requirements for WinRM:
  - pywinrm (pip install pywinrm)
  - Setup WinRM (https://github.com/ansible/ansible/blob/devel/examples/scripts/ConfigureRemotingForAnsible.ps1)
- The hosts propery can contain paterns (e.g.: wildcards)
- The `-i` for inventory also can accept scripts for dinamyc inventory (`ansible-playbook -i inventory.py playbook.yml`).

## Ansible Vault

It's possible to cipher all kind of variables and use it as, for example, passwords into the inventory file.

Create a new encrypted file:

```bash
ansible-vault create secrets
```

An example of the content of the file:

```yaml
ssh_password: P@ssw0rd!
sudo_password: r00t!!t00r
```

And it can be used in your inventory or inside the playbook:

```text
[webservers]
192.168.1.16 ansible_connection=ssh ansible_user=operator ansible_become_pass='{{sudo_password}}' ansible_ssh_pass='{{ssh_password}}'
```

