# Ansible-Lab
Practice ansible in windows(wsl as manage node) and Oracle virtual box (for worker node) with linux image
If we are working in windows system then install wsl and Oracle Virtual Box
So interaction will be made from wsl system to ubuntu/linux from oracle virtual image

Steps to follow:
	1.  Install & enable WSL2 (Ansible control runtime)
		a. PowerShell (Admin): wsl --install -d Ubuntu. 
		b. If you already had WSL1, set default to WSL2: wsl --set-default-version 2.
		c. Launch on ubuntu: sudo apt update && sudo apt upgrade -y
	2. Install Ansible inside WSL2
		a. sudo apt update
sudo apt install -y ansible openssh-client git python3-pip
ansible --version  ##check ansible version
sudo systemctl enable --now ssh
systemctl status ssh  ##check whether ssh system active
	3. VirtualBox networking (so WSL2 can reach your VMs)
		a. Goal: The WSL2 instance must be able to route to your VM IPs.
		b. Install any linux image in the box. Next steps are of network steps
		c. Host-Only Adapter (private lab)
			i. In virtual Box
				1) File --> Tools --> Network Manager --> create  ## It will create network for you
			ii. Linux system: Adapter1 -> Host-only Adapter.  ##Reachable from the Windows host
			iii. Adapter 2 --> NAT (for VM's outbound internet)
	4. Prepare the Ubuntu VMs (managed nodes)
		a. On Vm's run below commands
			i. # Make sure SSH server exists and is enabled
sudo apt update
sudo apt install -y openssh-server python3  ##Ansible copies Python modules to the remote host and executes them with Python
sudo systemctl enable --now ssh

# Confirm IP address (pick the Host-Only/Bridged interface)
ip a | grep -E "inet .*en|ens|eth"
# Or:
hostname -I
	5. Create SSH keys in WSL2 and distribute to VMs
		a. In WSL:
			i. ssh-keygen -t ed25519 -C "ansible-control@wsl"   # modern, fast; or use rsa -b 4096# Press Enter for defaults; optional passphrase (use ssh-agent if you set one) or
			ii. ssh-keygen -t rsa -b 4096  ## most command mode to use. 
			iii. ssh-copy-id youruser@<linux_ip>  ## copies your public key to remote host mean linux system /home/<user>/.ssh/authorized_keys
			iv. You can test it using: ssh 'youruser@<linux_ip>' 'hostname  -I && whoami'  ## output will have ip's and hostname
	6. Build a minimal Ansible project in WSL2
		a. mkdir -p ~/ansible-lab
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
			
		b. inventory.ini (managed nodes & connection vars)
			[webservers]
			Linux_Ubuntu ansible_host=192.168.56.101 ansible_user=ubuntu
	7. First connectivity test
		a. ansible all -m ping  ## this will show host name its ip and ping-pong message
	8. Ready-to-run sample playbook (Nginx + test page)
		a. Create site.yml:
			---
			- name: Configure Ubuntu web servers
			  hosts: webservers
			  gather_facts: false   # start without facts; we'll ensure Python first
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
		b. ansible-playbook site.yml  ## run this
		Test from your Windows browser:
			• http://<linux_ip>/
<img width="1375" height="3600" alt="image" src="https://github.com/user-attachments/assets/655aa68f-6b51-425a-860f-d6f36da7476a" />
