---
title: Create Gitea server using Ansible and docker
date: 2022-03-06 14:10:00 +0800
categories: [homelab, gitea]
tags: [gitea, docker, ansible, homelab]     # TAG names should always be lowercase
---

## About Gitea, docker, ssh workaround

Gitea is a lightweight git server you can self-host in your home lab using docker. 
There are some configurations you have to do to get it properly working with ssh since Gitea is running inside a docker, and it does not play easily with the server ssh etc.
I have created and Ansible playbook with the tasks to perform to create a git user on the server and configure it with the docker following the [official guide](https://docs.gitea.io/en-us/install-with-docker/) 
## Create the playbook 
 {% raw %}
```yaml
- name: Create git user on the server along with ssh keys
  ansible.builtin.user:
    name: git
    comment: git
    uid: 1500
    generate_ssh_key: yes
    ssh_key_type: rsa
    ssh_key_bits: 4096

- name: Print Debug info of the user created
  getent:
    database: passwd
    key: git
- debug:
    var: ansible_facts.getent_passwd


# - name: Generate /home/git/.ssh/id_rsa SSH RSA host key
#   command : sudo -u git ssh-keygen -q -t rsa -b 4096 -C "" -N "" -f /home/git/.ssh/id_rsa
#   args:
#     creates: /home/git/.ssh/id_rsa

- name: Copy public key to the client machine, workaround for authorized_key task
  fetch:    
    src: /home/git/.ssh/id_rsa.pub
    dest: /tmp/
    flat: yes

- name: Set authorized key took from file
  authorized_key:
    user: git
    state: present
    key: "{{ lookup('file', '/tmp/id_rsa.pub') }}"


- name: Set permission
  command : sudo -u git chmod 600 /home/git/.ssh/authorized_keys

- name: Insert a ssh command to authorized_keys
  lineinfile:
    path: /home/git/.ssh/authorized_keys
    line: command="/usr/local/bin/gitea --config=/data/gitea/conf/app.ini serv key-1",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty <user pubkey>


- name: Copy script to server
  ansible.builtin.copy:
    src: gitea
    dest: /usr/local/bin/gitea
    owner: "git"
    group: "git"
    mode: u+x

- name: Create gitea container
  docker_container:
    name: gitea
    image:  gitea/gitea
    state: started
    recreate: yes
    restart_policy: always
    env:
      USER_UID:  "1500"
      USER_GID:  "1500"      
    volumes:
      - /data/gitea:/data
      - /home/git/.ssh/:/data/git/.ssh
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    published_ports:
      - "3000:3000"
      - "2222:22"
```
{: .nolineno file="roles/gitea/tasks/main.yml" }
{% endraw %}

## Install the Gitea server 

After the playbook is executed, follow the webui instructions to install Gitea.
This will create the `app.ini` config file in the `data/gitea/gitea/conf/app.ini`.
Here is a redacted version of the file with the important configs for future reference.

```ini
APP_NAME = Git Server
RUN_MODE = prod
RUN_USER = git

[server]
APP_DATA_PATH    = /data/gitea
DOMAIN           = git.example.com
SSH_DOMAIN       = git.example.com
HTTP_PORT        = 3000
ROOT_URL         = http://git.example.com/
DISABLE_SSH      = false
SSH_PORT         = 22
SSH_LISTEN_PORT  = 22
LFS_START_SERVER = true
LFS_CONTENT_PATH = /data/git/lfs
LFS_JWT_SECRET   = 
OFFLINE_MODE     = false

```
{: .nolineno file="/data/gitea/gitea/conf/app.ini" }