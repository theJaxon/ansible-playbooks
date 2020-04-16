### Docker Playbook:
The initial goal was to write a playbook that can install docker on (CentOS, ~~Fedora~~ and Ubuntu).
However when comes to fedora, [no dnf specific module](https://github.com/ansible/ansible/issues/46963) exists for handling the repo addition part (this could've been done using either `command` or `shell`) modules but i just dropped the idea since the repo will be added each and every time unlike [yum_repository](https://docs.ansible.com/ansible/latest/modules/yum_repository_module.html) and [apt_repository](https://docs.ansible.com/ansible/latest/modules/apt_repository_module.html)


#### Preparing the testing environment:
- 2 vagrant machines ([CentOS 7](https://app.vagrantup.com/centos/boxes/7) and [Ubuntu Bionic](https://app.vagrantup.com/ubuntu/boxes/bionic64))
- ssh keys generated and public keys injected into `/home/vagrant/.ssh/authorized_keys`
- ansible master (WSL Ubuntu) using the private key to connect to these 2 machines 

#### Steps:
1- Generate the keys using `ssh-keygen`

```bash
 ssh-keygen -f /mnt/c/Users/Jaxon/Documents/Vagrant/DockerAnsibleTest/.ssh/id_rsa
```

2- Write a Vagrantfile that utilize [Multi Machine](https://www.vagrantup.com/docs/multi-machine/) feature and use an inline shell provisioner to append the public key to the authorized_keys file.

<details><summary>Vagrantfile</summary>
<p>

```Ruby

Vagrant.configure("2") do |config|
    config.vm.synced_folder "./", "/vagrant", type: "virtualbox", disabled: false
    config.vm.box_check_update = false
    config.vbguest.auto_update = false # disable OS related updates on boot
    config.vm.network "forwarded_port", guest: 22, host: 22, # SSH
    auto_correct: true # fix port collisions

    config.vm.define "CentOS7" do |centOS7|
        centOS7.vm.box = "centos/7"
        centOS7.vm.hostname = "CentOS7"
        centOS7.vm.network "public_network", ip: "192.168.1.10"
        centOS7.vm.provision "shell", inline: "echo CentOS 7 && cat /vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys"
    end 

    config.vm.define "Ubuntu" do |ubuntu|
        ubuntu.vm.box = "ubuntu/bionic64"
        ubuntu.vm.hostname = "Ubuntu"
        ubuntu.vm.network "public_network", ip: "192.168.2.10"
        ubuntu.vm.provision "shell", inline: "echo Ubuntu 7 && cat /vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys"
    end

end
```

* vbguest plugin needs to be installed 

` vagrant plugin install vagrant-vbguest `

</p>
</details>


3- Write ansible's inventory file

<details><summary>inventory</summary>
<p>

```

[centos]
server-name-centos ansible_host=192.168.1.10 ansible_user=vagrant ansible_port=22

[ubuntu]
server-name-ubuntu ansible_host=127.0.0.1 ansible_user=vagrant ansible_port=2203

```

</p>
</details>

4- write the main playbook that will be used to execute all other ansible scripts

<details><summary>main.yml</summary>
<p>

```yaml

---
- hosts: all
  remote_user: vagrant
  become: true
  roles:
    - Docker
    
```

</p>
</details>

5- write the main.yml file located in /tasks directory, in my case i used it to reference 2 other yml files, these 2 files will be executed depending on the type of the OS (if it's either CentOS or Ubuntu)

<details><summary>/tasks/main.yml</summary>
<p>

```yaml

---
# tasks file for Docker

# Ubuntu Tasks
- include_tasks: ./Ubuntu.yml
  when: ansible_distribution == "Ubuntu"


# CentOS tasks
- include_tasks: ./CentOS.yml
  when: ansible_distribution == "CentOS"


```

</p>
</details>

6- write var file `/vars` that contains names of packages that will be installed or removed

<details><summary>/vars/main.yml</summary>
<p>

```yaml

---
# vars file for Docker
CentosRM:
  - docker
  - docker-client 
  - docker-common
  - docker-latest
  - docker-latest-logrotate
  - docker-logrotate
  - docker-engine

UbuntuPackages:
  - apt-transport-https 
  - ca-certificates 
  - curl 
  - gnupg-agent 
  - software-properties-common

DockerCentOSPackages:
  - docker-ce 
  - docker-ce-cli 
  - containerd.io

```

</p>
</details>

7- write the steps for installing docker on CentOS and Ubuntu

<details><summary>/tasks/CentOS</summary>
<p>

```yaml

- name: remove old docker version 
  package: 
    name: "{{ CentosRM }}"
    state: absent 

- name: install yum-utils
  package: 
    name: yum-utils
    state: present

- name: Add docker CE repository
  get_url:
      url: https://download.docker.com/linux/centos/docker-ce.repo
      dest: /etc/yum.repos.d/docer-ce.repo

- name: add gpg key for the repo
  rpm_key:
    key: https://download.docker.com/linux/centos/gpg
    fingerprint: 060A61C51B558A7F742B77AAC52FEB6B621E9F35
    state: present


- name: Install docker engine
  package: 
    name: "{{ DockerCentOSPackages }}"
    state: present 

- name: start docker & enable it on boot 
  systemd:
    name: docker
    enabled: yes 
    state: started 

```

</p>
</details>

<details><summary>/tasks/Ubuntu</summary>
<p>

```yaml

- name: remove old docker version [Ubuntu]
  package: 
    name: 
      - docker 
      - docker-engine
      - docker.io
      - containerd
      - runc
    state: absent 

- name: apt-get update 
  apt:
    update_cache: yes

- name: install required Ubuntu packages 
  package:
    name: "{{ UbuntuPackages }}"

- name: Add docker gpg key 
  apt_key:
    id: 9DC858229FC7DD38854AE2D88D81803C0EBFCD88
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: add repo docker on ubuntu
  apt_repository:
    repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu "{{ansible_distribution_release}}" stable
    state: present

- name: apt-get update 
  apt:
    update_cache: yes

- name: install docker on Ubuntu
  package: 
    name: 
      - docker-ce 
      - docker-ce-cli 
      - containerd.io 
    state: present

```

</p>
</details>

8- Use ansible to run the main playbook 

```bash

ssh-agent bash 

ssh-add ~/.ssh/id_rsa

ansible-playbook -i inventory main.yml

```