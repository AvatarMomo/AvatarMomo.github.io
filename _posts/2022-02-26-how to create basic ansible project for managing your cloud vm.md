---
title: How to create a simple ansible project for setting up your cloud vm instance
date: 2022-02-26 14:10:00 +0000
categories: [Server, Ansible]
tags: [Server, Ansible, Oracle Cloud server, playbook]     # TAG names should always be lowercase
---

## Requirements
1. Create a server in your favourive cloud provider and make sure ssh is working.
2. [Install ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#) 

## Setting up simple ansible project
Today I tried to setup a VM in Oracle Private Cloud using their free tier offerings.
This is an easy way to get a server up and running. Check out what is provided under 
I am also documenting how to setup your basic ansible project to manage your VM.

Here I am using Oracle Linux 9 to setup my vm.

This is the basic overview of the files you need to create the basic Infrastructure as a code setup using ansible.

### folder overview of the ansible project

```
📦oracle_cloud_iac
 ┣ 📂group_vars
 ┃ ┗ 📂all
 ┃ ┃ ┗ 📜vars.yml
 ┣ 📂tasks
 ┃ ┗ 📜setup.yml
 ┣ 📜ansible.cfg
 ┣ 📜hosts
 ┗ 📜tasks_playbook.yml
 ```
 {:file="folder tree" }


### Setting up hosts
hosts is where you configure your connection to your VM.

> Make sure to edit this file.
{: .prompt-warning }
```ini
[server]
opc_server ansible_host=INSTANCE_PUBLIC_IP ansible_port=22 ansible_user=opc ansible_connection=ssh ansible_ssh_private_key_file=/home/user/.ssh/id_rsa

```
{: .nolineno file="hosts" }

### Setting up ansible.cfg
This is ansible where ansible specific settings are specified.

```ini
[defaults]
INVENTORY = hosts

[ssh_connections]
pipelining = true
```
{: .nolineno file="ansible.cfg" }

### Setting up vars.yml
The vars.yml file is used to store variable that you can access from your playbooks.
Here I am using it to keep a list of packages that I would like to install
```yaml
username: opc 
packages_essential:
  - vim
  - oracle-epel-release-el8
  - emacs
  - tmux
  - git
  - python3-pip
  - dnf-utils
  - zip 
  - unzip

```
{: .nolineno file="group_vars/all/vars.yml" }

### Setting up simple playbook
I am going to create a playbook that will execute commands as root on my configured server.

```yaml
---
- hosts: opc_server 
  become: yes 

  tasks:

    - import_tasks: tasks/setup.yml

```
{: .nolineno file="tasks_playbook.yml" }


### Add tasks 
In this task I have put some basic commands like updating and installing packages including docker.

 {% raw %}

```yaml
- name: Update packages
  dnf:
    name: "*"
    state: latest

- name: Install packages
  dnf:
    name: "{{packages_essential}}"
    state: latest

- name: Install Docker Module for Python
  pip:
    name: docker

- name: Add docker repo
  shell:  'dnf config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo'


- name: Install Docker
  dnf:
    name: "docker-ce"
    state: latest

- name: Enable service docker and ensure it is not masked
  systemd:
    name: docker
    enabled: yes
    masked: no

- name: Start docker service
  systemd:
    state: started
    name: docker

```
{: .nolineno file="tasks/setup.yml" }

{% endraw %}

## Running the ansible playbook
Finally run the ansible playbook from your terminal

```console
$ ansible-playbook tasks_playbook.yml
```
If there are errors with any tasks playbook will stop execution
Wish you all the best. :smirk: