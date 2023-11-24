# ansible
- In ansible-role, we are not writing the all code in single file. we keep in separate files
- Ansible role is a structured form of ansible playbook

---

```
 mkdir ansible-role
 cd ansible-role
```
### creating ansible role named lamp
```
  $ansible-galaxy init lamp
  - Role lamp was created successfully
```
```
$ tree
.
└── lamp
 ├── README.md
 ├── defaults
 │ └── main.yml
 ├── handlers
 │ └── main.yml
 ├── meta
 │ └── main.yml
 ├── tasks
 │ └── main.yml
 ├── tests
 │ ├── inventory
 │ └── test.yml
 └── vars
 └── main.yml
```
```
 $ansible-galaxy init updation
 Role updation was created successfully
```
```

 ll
drwxr-xr-x 8 aravind aravind 4096 Nov 24 21:01 lamp/
drwxr-xr-x 8 aravind aravind 4096 Nov 24 21:04 updation/
```
---

Server updation tasks

- Tasks that are written in ansible role must be written in ./updation/tasks/main.yml

---

```
cat updation/tasks/main.yml
```
```
---

- name: "updating the server"
yum:
 name: "*"
 state: latest
```
---

### Calling the role from any ansible project
Creating a new project directory

```
 mkdir ansible-project
```
```
 touch host main.yml aws.pem
```
```
 $cat host
[amazon]
172.31.10.38 ansible_user="ec2-user" ansible_port=22 ansible_ssh_private_key_file="aws.pem"
```
---

#### Ping to all server
```
ansible -i hosts all -m ping
```
---

#### Creating Ansible configuration file  in the project directory
On running the ansible playbook using role ansible use this .cfg file

ref: [﻿riptutorial.com/ansible/example/21992/ansible-cfg](https://riptutorial.com/ansible/example/21992/ansible-cfg) 



```
vi ansible.cfg
```
```
[defaults]
fork = 10
roles_path = /home/aravindcm/ansible-role/

```
```
$ansible --version

ansible 2.10.8
config file = /home/aravind/ansible-project/ansible.cfg
configured module search path = ['/home/aravind/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
ansible python module location = /usr/lib/python3/dist-packages/ansible
executable location = /usr/bin/ansible
python version = 3.10.12 (main, Jun 11 2023, 05:26:28) [GCC 11.4.0]
```
---

#### Creating the Playbook
```
$ vi  main.yml
---
- name: " Wordpress installion"
  host: amazon
  role:
    - updation
```
```
ansible-playbook -i host main.yml
```
This will run the Ansible playbook and also run the task we wrote on the updation role

---

In roles variables can store either in defaults/main.yml or in var/main.yml 

higher priority vars is in ./var/main.yml  

---

```
~/ansible-role/lamp$ vi vars/main.yml
```
```
---
# vars file for lamp
mysql_passwd: "root#1234d"
db_name: "my_db"
db_user: "my_user"
db_user_passwd: "sc#2345"
domain_name: "blog.avaincm.live"
httpd_owner: "apache"
httpd_group: "apache"
httpd_port: 8080
repo_url: "https://github.com/aravindcm95/aws-elb-site.git"
wp_url: https://wordpress.org/wordpress-6.3.2.tar.gz               # wordpress url
packages:
  - mariadb105-server
  - python3-pip
  - httpd
  - php
  - git
```
---

-  Templates of the playbook is stored in `ansible-role/lamp/templates` 
- Normal files of Ansible are stored in `ansible-role/lamp/files` 
```
$ tree
.
├── README.md
├── defaults
│   └── main.yml
├── files
│   ├── test.html
│   └── test.php
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   ├── httpd.conf.tmpl
│   └── virtual.conf.tmpl
├── test.php}
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```
---

~/ansible-role/lamp$ cat tasks/main.yml

```
---
# tasks file for lamp

- name: "packages installing"
    yum:
    name: "{{packages}}"
    state: present
- name: "db- starting/ enabling"
  service:
    name: mariadb
    state: started
    enabled: true
- name: " installing python3 mysql module pyMySQL"
  pip:
    name: " pyMySQL"
    state: present
- name: "Setting root password for maria-db-server"
  ignore_errors: true
  mysql_user:
    login_user: "root"
    login_password: ""
    login_unix_socket: /var/lib/mysql/mysql.sock     # mysql socket file location in remote server
    user: "root"
    password: "{{ mysql_passwd }}"
- name: "removing anonymous users"
  mysql_user:
    login_user: "root"
    login_password: "{{ mysql_passwd }}"
    login_unix_socket: /var/lib/mysql/mysql.sock
    user: ""
    state: absent
- name: " removing test databases "
  mysql_db:
    login_user: "root"
    login_password: "{{ mysql_passwd }}"
    login_unix_socket: /var/lib/mysql/mysql.sock
    name: "test"
    state: absent
- name: "creating Initial databases"
  mysql_db:
    login_user: "root"
    login_password: "{{ mysql_passwd }}"
    login_unix_socket: /var/lib/mysql/mysql.sock
    name: "{{ db_name }}"
    state: present
- name:  "Creating User called {{ db_user }}"
  community.mysql.mysql_user:
    login_user: "root"
    login_password: "{{ mysql_passwd }}"
    login_unix_socket: /var/lib/mysql/mysql.sock
    user: "{{ db_user }}"
    password: "{{ db_user_passwd  }}"
    priv: '{{ db_name }}.*:ALL'
    state: present



- name: " creating httpd.conf from template"
   template:
     src: httpd.conf.j2  #no need to specify the path
     dest: /etc/httpd/conf/httpd.conf
   register: config_status
 - name: "Creating virtual config file for {{ domain_name}}"
   template:
     src: virtualhost.conf.j2 #no need to specify the path
     dest: /etc/httpd/conf.d/default.conf
     owner: "{{ httpd_owner }}"
     group: "{{ httpd_group }}"
   register: vhost_status
 - name: "creating document root"
   file:
     path: /var/www/html/_default/
     state: directory
     owner: "{{ httpd_owner }}"
     group: "{{ httpd_group }}"
 - name: "copy the test page"
   copy:
     src: "{{ item}}"
     dest: /var/www/html/_default/
     owner: "{{ httpd_owner }}"
     group: "{{ httpd_group }}"
   with_item:
     - test.html
     - test.php
 - name: "copy repository"
   git:
     repo: "{{ repo_url }}"
     dest: "/var/website"
   register: git_status
 - name: " copy coloned site to documentRoot"
   when: git_status.changed == true
   copy:
     src: "/var/website/"
     dest: "/var/www/html/_default"
     remote_src: true
     owner: "{{ httpd_owner }}"
     group: "{{ httpd_group }}"
 - name: " restarting apache"
   when: config_status.changed == true or vhost_status.changed == true
   service:
     name: httpd
     state: restarted
     enabled: true
 - name: " restarting php"
   when: git_status.changed == true or vhost_status.changed == true
   service:
     name: php-fpm
     state: restarted
     enabled: true
```
---

goto ansible project and edit main.yml

```
$ vi  main.yml
---
- name: " Wordpress installion"
  host: amazon
  role:
    - updation
    - lamp
```
```
$ ansible-playbook -i host main.yml
```
---



