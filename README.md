# Jira data center project

### Project Documentation link
https://eu.docworkspace.com/d/sIITam9SoAtCIj8QG?sa=601.1037

# 3 EC2 instances
# name them, controlnode, workernode1 and workernode2
# connect to all the 3 servers
# assign hostname to the servers respectively

        sudo hostnamectl set-hostname controlnode

# create a common user across the 3 servers 

       sudo adduser <username>

# assign passwd to all 3 users

      sudo passwd <username>

# add user to the suoders group across 3 servers

      echo "ansible ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansible

# assign password authentication to all 3 servers

      sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config

# uncomment permitrootlogin in the /etc/ssh/sshd_config file

      sudo sed -i 's/^#PermitRootLogin/PermitRootLogin/' /etc/ssh/sshd_config

# restart the sshd service

       sudo systemctl restart sshd

## do the above in all the 3 nodes

        #!/bin/bash
        #make sure you edit the hostname with correct nodename 
        sudo hostnamectl set-hostname Ansible-control
        sudo adduser ansible
        sudo echo "ansible:ansible" | chpasswd
        echo "ansible ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansible
        sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
        sudo sed -i 's/^#PermitRootLogin/PermitRootLogin/' /etc/ssh/sshd_config
        sudo systemctl restart sshd

# after doing that, try to connect to the workernodes via the controlnode

         ssh ansible@private-ip

## In the controlnode, switch to user, ansible

         sudo su - ansible

# As user ansible, navigate to the home working directory
  of ansible

           cd ~

# verify if you are in the home of ansible

          pwd
 
# Generate a key-pair

          ssh-keygen -t rsa

# verify keypair

          ls .ssh

# assign executable permissions to  the .ssh file so as to copy the key to workernodes

          sudo chmod 700 /home/ansible/.ssh

# copy the key-pair to workernodes (do same for each)

         ssh-copy-id ansible@private-ip

# after doing that, try to ssh to workernoders without key-pair

# Install ansible in the control node

         sudo amazon-linux-extras install ansible2 -y

# Verrify if ansible is installed

         ansible --version

## Either login to the workernodes via controlnode or directly and install docker

         sudo amazon-linux-extras install docker -y

# start docker

         sudo service docker start

# enable docker

        sudo systemctl enable docker
 
# verify if docker is installed

        sudo docker run hello-world
        sudo systemctl status docker

## simple script for the installation of docker

      #!/bin/bash
      amazon-linux-extras install docker -y
      sudo service docker start
      sudo systemctl enable docker
      sudo docker run hello-world
      sudo systemctl status docker

### You can run the below script to configure the workernodes and install docker

      #!/bin/bash
      #make sure you edit the hostname with correct nodename 
      #sudo hostnamectl set-hostname node1
      sudo adduser ansible
      sudo echo "ansible:ansible" | chpasswd
      echo "ansible ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansible
      sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
      sudo sed -i 's/^#PermitRootLogin/PermitRootLogin/' /etc/ssh/sshd_config
      sudo systemctl restart sshd
      sudo yum install docker -y
      sudo service docker start
      sudo systemctl enable docker
      sudo docker run hello-world
      sudo systemctl status docker
     


## Go back to the controlnode 

# navigate to /etc/ansible and define your workernodes

           cd /etc/ansible

# vi to the hosts file

           vi hosts

# uncomment ##[webservers] and add your wockernodes private-ip under

         [webservers]
         ## alpha.example.org
         ## beta.example.org
         ## 192.168.1.100
         ## 192.168.1.110
         [added worker-nodes private-IPs]  

# verify if you can ping

         ansible all -m ping 

# create a directory jira-docker

         mkdir jira-docker

# create a file docker-compose.yml in jira-docker directory

         touch jira-docker/docker-compose.yml

# navigate to the file and paste the below docker compose file code

              vi docker-compose.yml


version: '3'
services:
  jira:
    image: atlassian/jira-software:9.9.1-jdk11  # Update to the specific Jira Software version 9.9.1-jdk11
    ports:
      - "8080:8080"
    volumes:
      - jira-data:/var/atlassian/jira
    environment:
      - ATL_JDBC_URL=jdbc:postgresql://db/jiradb
      - ATL_JDBC_USER=jirauser
      - ATL_JDBC_PASSWORD=jirapassword
  db:
    image: postgres:9.6
    environment:
      - POSTGRES_DB=jiradb
      - POSTGRES_USER=jirauser
      - POSTGRES_PASSWORD=jirapassword
    volumes:
      - db-data:/var/lib/postgresql/data
volumes:
  jira-data:
  db-data:

## create a second directory ansible-playbooks

          mkdir ansible-playbooks

# navigate to it and touch a file deploy_jira.yml

          cd ansible-playbooks

# Create file          
        
            touch deploy_jira.yml

# Navigate to the file
             
             vi deploy_jira.yml

# copy the below playbook code to it

---
- name: Deploy Jira Data Center
  hosts: webservers
  become: true
  become_user: root
  tasks:
    - name: Ensure Docker is installed
      yum:
        name: docker
        state: present
      tags: docker
    - name: Ensure Docker service is started
      service:
        name: docker
        state: started
        enabled: yes
      tags: docker
    - name: Check Docker service status
      command: systemctl status docker
      register: docker_status
      tags: debug
    - name: Print Docker service status
      debug:
        var: docker_status.stdout
      tags: debug
    - name: Ensure Docker Compose is installed
      get_url:
        url: https://github.com/docker/compose/releases/download/1.29.2/docker-compose-{{ ansible_system }}-{{ ansible_architecture }}
        dest: /usr/local/bin/docker-compose
        mode: '0755'
      tags: docker_compose
    - name: Check Docker version
      command: docker --version
      register: docker_version
      tags: debug
    - name: Print Docker version
      debug:
        var: docker_version.stdout
      tags: debug
    - name: Check Docker Compose version
      command: /usr/local/bin/docker-compose --version
      register: docker_compose_version
      tags: debug
    - name: Print Docker Compose version
      debug:
        var: docker_compose_version.stdout
      tags: debug
    - name: Create directory for Jira Docker Compose file
      file:
        path: /opt/jira
        state: directory
        mode: '0755'
      tags: jira
    - name: Copy Docker Compose file to the server
      copy:
        src: ~/jira-docker/docker-compose.yml
        dest: /opt/jira/docker-compose.yml
      tags: jira
    - name: Start Jira using Docker Compose
      command: /usr/local/bin/docker-compose up -d
      args:
        chdir: /opt/jira
      register: compose_output
      tags: jira
    - name: Print Docker Compose command output
      debug:
        var: compose_output
      tags: debug


## Run playbook in the directory where playbook is located

         ansible-playbook deploy_jira.yml

## verify if application is running

## public_ip:8080 # any workernode public-ip                 


# NOTE:
host-ip - ansible_ssh_pass=ansible --> ignores passwd overlap


## Load balacers

-create load balacers for the app

     Name = Ansible-LB
     
- select SG

## Target Groups

-creat Target groups to target the all
- edit the HTTP trffic port number with correct one


## Docker-compose.yml file

version: '3'
services:
  jira:
    image: atlassian/jira-software:9.9.1-jdk11  # Update to the specific Jira Software version 9.9.1-jdk11
    ports:
      - "8080:8080"
    volumes:
      - jira-data:/var/atlassian/jira
    environment:
      - ATL_JDBC_URL=jdbc:postgresql://db/jiradb
      - ATL_JDBC_USER=jirauser
      - ATL_JDBC_PASSWORD=jirapassword
  db:
    image: postgres:9.6
    environment:
      - POSTGRES_DB=jiradb
      - POSTGRES_USER=jirauser
      - POSTGRES_PASSWORD=jirapassword
    volumes:
      - db-data:/var/lib/postgresql/data
volumes:
  jira-data:
  db-data:

git clone https://emmanuel40@bitbucket.org/emmanuel_workspace/jira-repo.git


## Deployment playbook for 9.9.1 jira version

---
- name: Deploy Jira Data Center
  hosts: webservers
  become: true
  become_user: root
  tasks:
    - name: Ensure Docker is installed
      yum:
        name: docker
        state: present
      tags: docker
    - name: Ensure Docker service is started
      service:
        name: docker
        state: started
        enabled: yes
      tags: docker
    - name: Check Docker service status
      command: systemctl status docker
      register: docker_status
      tags: debug
    - name: Print Docker service status
      debug:
        var: docker_status.stdout
      tags: debug
    - name: Ensure Docker Compose is installed
      get_url:
        url: https://github.com/docker/compose/releases/download/1.29.2/docker-compose-{{ ansible_system }}-{{ ansible_architecture }}
        dest: /usr/local/bin/docker-compose
        mode: '0755'
      tags: docker_compose
    - name: Check Docker version
      command: docker --version
      register: docker_version
      tags: debug
    - name: Print Docker version
      debug:
        var: docker_version.stdout
      tags: debug
    - name: Check Docker Compose version
      command: /usr/local/bin/docker-compose --version
      register: docker_compose_version
      tags: debug
    - name: Print Docker Compose version
      debug:
        var: docker_compose_version.stdout
      tags: debug
    - name: Create directory for Jira Docker Compose file
      file:
        path: /opt/jira
        state: directory
        mode: '0755'
      tags: jira
    - name: Copy Docker Compose file to the server
      copy:
        src: ~/jira-docker/docker-compose.yml
        dest: /opt/jira/docker-compose.yml
      tags: jira
    - name: Start Jira using Docker Compose
      command: /usr/local/bin/docker-compose up -d
      args:
        chdir: /opt/jira
      register: compose_output
      tags: jira
    - name: Print Docker Compose command output
      debug:
        var: compose_output
      tags: debug





## Deployment Playbook of latest version

---
- name: Deploy Jira Data Center
  hosts: webservers
  become: true
  become_user: root
  tasks:
    - name: Ensure Docker is installed
      yum:
        name: docker
        state: present
      tags: docker
    - name: Ensure Docker service is started
      service:
        name: docker
        state: started
        enabled: yes
      tags: docker
    - name: Check Docker service status
      command: systemctl status docker
      register: docker_status
      changed_when: false
      failed_when: docker_status.rc != 0
      tags: debug
    - name: Print Docker service status
      debug:
        var: docker_status.stdout
      tags: debug
    - name: Ensure Docker Compose is installed
      get_url:
        url: https://github.com/docker/compose/releases/download/1.29.2/docker-compose-{{ ansible_system }}-{{ ansible_architecture }}
        dest: /usr/local/bin/docker-compose
        mode: '0755'
      tags: docker_compose
    - name: Check Docker version
      command: docker --version
      register: docker_version
      changed_when: false
      failed_when: docker_version.rc != 0
      tags: debug
    - name: Print Docker version
      debug:
        var: docker_version.stdout
      tags: debug
    - name: Check Docker Compose version
      command: /usr/local/bin/docker-compose --version
      register: docker_compose_version
      changed_when: false
      failed_when: docker_compose_version.rc != 0
      tags: debug
    - name: Print Docker Compose version
      debug:
        var: docker_compose_version.stdout
      tags: debug
    - name: Create directory for Jira Docker Compose file
      file:
        path: /opt/jira
        state: directory
        mode: '0755'
      tags: jira
    - name: Copy Docker Compose file to the server
      copy:
        src: ~/jira-docker/docker-compose.yml
        dest: /opt/jira/docker-compose.yml
        mode: '0644'
      tags: jira
    - name: Stop Jira using Docker Compose
      command: /usr/local/bin/docker-compose down
      args:
        chdir: /opt/jira
      register: compose_down_output
      tags: jira
    - name: Print Docker Compose down command output
      debug:
        var: compose_down_output.stdout
      tags: debug
    - name: Update Jira Software image version to latest
      replace:
        path: /opt/jira/docker-compose.yml
        regexp: 'atlassian/jira-software:.*'
        replace: 'atlassian/jira-software:latest'
      tags: jira
    - name: Pull latest Jira Software image
      command: /usr/local/bin/docker-compose pull
      args:
        chdir: /opt/jira
      register: compose_pull_output
      tags: jira
    - name: Print Docker Compose pull command output
      debug:
        var: compose_pull_output.stdout
      tags: debug
    - name: Start Jira using Docker Compose
      command: /usr/local/bin/docker-compose up -d
      args:
        chdir: /opt/jira
      register: compose_up_output
      tags: jira
    - name: Print Docker Compose up command output
      debug:
        var: compose_up_output.stdout
      tags: debug





## Script to uninstall Docker

         sudo yum purge -y docker-ce docker-ce-cli containerd.io
         sudo rm -rf /var/lib/docker
         sudo rm -rf /var/lib/containerd


https://www.atlassian.com/software/compass







