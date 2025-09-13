# Ansible-Lab setup in windows
Practice ansible in windows (wsl as manage node) and Oracle virtual box (for worker node) with linux image (with passwordless login).

If we are working in windows system then install wsl and Oracle Virtual Box
So interaction will be made from wsl system to ubuntu/linux from oracle virtual image

Steps to follow:
## Install & enable WSL2 (Ansible control runtime
### PowerShell (Admin):
```bash
 wsl --install -d Ubuntu.
``` 
### If you already had WSL1, set default to WSL2:
```bash
wsl --set-default-version 2
```
## Launch on ubuntu:
```bash
sudo apt update && sudo apt upgrade -y
```
### Install Ansible inside WSL2
```bash		
sudo apt update
sudo apt install -y ansible openssh-client git python3-pip 
ansible --version  ## check ansible version
sudo systemctl enable --now ssh
systemctl status ssh  ## check whether ssh system active
```
## VirtualBox networking (so WSL2 can reach your VMs)
#### Goal: The WSL2 instance must be able to route to your VM IPs(Orcle virutal box vm).
#### Install any linux image in the box [Linux_Images](https://www.linuxvmimages.com/images/ubuntuserver-2404/). 
### Network setups in virutal VM
- Host-Only Adapter (private lab)
	- In virtual Box
		- File --> Tools --> Network Manager --> create  ## It will create network for you
	- Linux system: Adapter1 -> Host-only Adapter.  ##Reachable from the Windows host
	- Adapter 2 --> NAT (for VM's outbound internet)
## Prepare the Ubuntu VMs (managed nodes)
### On Vm's: run below commands
Make sure SSH server exists and is enabled
```bash
sudo apt update
sudo apt install -y openssh-server python3  ## Ansible copies Python modules to the remote host and executes them with Python
sudo systemctl enable --now ssh
```
Confirm IP address (pick the Host-Only/Bridged interface)
```bash
ip a | grep -E "inet .*en|ens|eth"
# Or:
hostname -I
```
## Create SSH keys in WSL2 and distribute to VMs
### In WSL:
```bash
ssh-keygen -t ed25519 -C "ansible-control@wsl"   # modern, fast; or use rsa -b 4096  ## Press Enter for defaults; optional passphrase (use ssh-agent if you set one)
```
#### OR
```bash
ssh-keygen -t rsa -b 4096  ## most command mode to use. 
```
```bash
ssh-copy-id youruser@<linux_ip>  ## copies your public key to remote host mean linux system /home/<user>/.ssh/authorized_keys
```
- You can test it using: 
```bash
ssh 'youruser@<linux_ip>' 'hostname  -I && whoami'  ## output will have ip's and hostname
```
## Build a minimal Ansible project in WSL2
```bash		
mkdir -p ~/ansible-lab
cd ~/ansible-lab
ansible.cfg (controller behavior)
			[defaults]
			inventory = ./inventory.ini
			host_key_checking = False
			interpreter_python = auto_silent
			
			[privilege_escalation]
			become = True
			become_method = sudo
			become_ask_pass = False
			
inventory.ini (managed nodes & connection vars)
			[webservers]
			Linux_Ubuntu ansible_host=192.168.56.101 ansible_user=ubuntu
```
## First connectivity test
```bash
ansible all -m ping  ## this will show host name its ip and ping-pong message
```
## Ready-to-run sample playbook (Nginx + test page)
Create site.yml:
```bash
---
- name: Configure Ubuntu web servers
    hosts: webservers
    gather_facts: false   ## start without facts; we'll ensure Python first
    become: true

    pre_tasks:
    - name: Ensure Python exists (for Ansible modules)
        raw: |
        test -e /usr/bin/python3 || (apt-get update -y && apt-get install -y python3)
        changed_when: false

    - name: Gather facts now that Python is present
        setup:

    tasks:
    #- name: Update apt cache (safe & idempotent)
    # apt:
    #  update_cache: true
    #  cache_valid_time: 3600

    - name: Install Nginx
        apt:
        name: nginx
        state: present

    - name: Ensure Nginx is enabled and running
        service:
        name: nginx
        state: started
        enabled: true
    - name: Create index.html
        copy:
        dest: /var/www/html/index.html
        content: |
            <!doctype html>
            <html>
            <head>
            <meta charset="utf-8" />
            <title>Ansible Deployed</title>
            </head>
            <body>
            <h1>✅ Deployed by Ansible</h1>
            <p>Host: {{ inventory_hostname }}</p>
            <p>Time: {{ ansible_date_time.iso8601 }}</p>
            </body>
            </html>
        owner: www-data
        group: www-data
        mode: '0644'

    - name: Open HTTP in UFW if UFW is active (non-fatal if absent)
        block:
        - name: Check UFW status
            command: ufw status
            register: ufw_status
            changed_when: false
            failed_when: false

        - name: Allow HTTP if UFW active
            ufw:
            rule: allow
            name: 'Nginx Full'
            when: "'Status: active' in ufw_status.stdout"
    rescue:
        - debug:
            msg: "UFW not present or command failed; skipping firewall config."
```
```bash
ansible-playbook site.yml  ## run this
```
Test from your Windows browser:

	• http://<linux_ip>/
