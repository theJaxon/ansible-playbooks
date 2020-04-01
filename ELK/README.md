### Vagrant Centos/7 setup:
[![forthebadge](https://forthebadge.com/images/badges/designed-in-ms-paint.svg)](https://forthebadge.com)
[![forthebadge](https://forthebadge.com/images/badges/check-it-out.svg)](https://forthebadge.com)

#### [Port forwarding](https://www.vagrantup.com/docs/networking/forwarded_ports.html) in Vagrantfile:
```ruby
  config.vm.box = "centos/7"
  config.vm.network "forwarded_port", guest: 5044, host: 5044
  config.vm.network "forwarded_port", guest: 5601, host: 5601 # Kibana
  config.vm.network "forwarded_port", guest: 8080, host: 8080 # Jenkins
  config.vm.network "forwarded_port", guest: 9200, host: 9200 # Elasticsearch
```

* Configure the vagrant instance to have at minimum `5 GB of RAM`, see [prerequisites](https://elk-docker.readthedocs.io/#prerequisites) for the elk stack image

#### Ansible installation:
```bash
sudo yum update && sudo yum install python3 python-pip -y
sudo pip install docker-py # used when pulling the image with ansible's docker image module
sudo pip3 install ansible
```
> Command used to run the ELK-play.yaml locally was: 

$ ` ansible-playbook --connection=local  ELK-play.yaml -e 'ansible_python_interpreter=/usr/bin/python'`

* There's an issue with CentOS when using python3 

* sebp/elk image is quite large `879.36 MB` so it takes some time to pull the image, be patient por favor.

#### ELK-play.yaml:
```yaml
---
- hosts: 127.0.0.1
  become: true
  tasks:
  - name: install requirements for both jenkins & docker
    package:
      name:
        - yum-utils
        - device-mapper-persistent-data
        - lvm2
        - wget
        - java-1.8.0-openjdk
      state: present

  - name: setting virtual memory
    shell: >
      sysctl -w vm.max_map_count=262144

  - name: Add docker & jenkins repos to /etc/yum.repos.d/
    get_url:
      url: "{{item.url}}"
      dest: /etc/yum.repos.d/{{item.dest}}
    with_items:
      - {url: https://download.docker.com/linux/centos/docker-ce.repo , dest: docker-ce.repo }
      - {url: http://pkg.jenkins-ci.org/redhat-stable/jenkins.repo, dest: jenkins.repo }

  - name: install docker from the added repo
    package:
      name:
        - docker-ce
        - docker-ce-cli
        - containerd.io
      state: present

  - name: start docker service & enable it on boot
    service:
      name: docker
      state: started
      enabled: yes

  - name: pull the latest elk image from dockerhub /sebp/elk/
    docker_image:
      name: sebp/elk
      source: pull

  - name: start a container from the pulled elk image
    docker_container:
      name: elk
      image: sebp/elk
      ports:
        - "5601:5601"
        - "9200:9200"
        - "5044:5044"
      state: started

  - name: pull jenkins image from jenkins/jenkins 
    docker_image:
      name: jenkins/jenkins
      source: pull

  - name: start a container from jenkins pulled image 
    docker_container:
      name: jenkins
      image: jenkins/jenkins
      ports:
        - "8080:8080"
        - "5000:5000"
      state: started
```
- After pulling jenkins the plugin won't install because we need to first type the initial admin password to jenkins
Attach to the jenkins container after pulling it
$ `sudo docker container exec -it jenkins /bin/bash`
$ `cat /var/jenkins_home/secret/initialAdminPassword`

### Displaying Kibana dashboard:
- By default when you type `http://localhost:5601` nothing will appear because you need to create a [dummy log](https://elk-docker.readthedocs.io/#creating-dummy-log-entry) first so:

$ `vagrant ssh default` from another terminal and do the following:
```bash
sudo docker exec -it elk /bin/bash # interactive docker terminal
$# /opt/logstash/bin/logstash --path.data /tmp/logstash/data -e 'input { stdin { } } output { elasticsearch { hosts => ["localhost"] } }'
```
- Now you should see Kibana's dashboard on `http://localhost:5601`
- refer to the [elk-docker documentation](https://elk-docker.readthedocs.io) for more info 

##### Ansible modules used:
- [docker_container](https://docs.ansible.com/ansible/latest/modules/docker_container_module.html)
- [docker_image](https://docs.ansible.com/ansible/latest/modules/docker_image_module.html)
- [get_url](https://docs.ansible.com/ansible/latest/modules/get_url_module.html)
- [jenkins_plugin](https://docs.ansible.com/ansible/latest/modules/jenkins_plugin_module.html) 
- [Package](https://docs.ansible.com/ansible/latest/modules/package_module.html)
- [Shell](https://docs.ansible.com/ansible/latest/modules/shell_module.html)
- [Service](https://docs.ansible.com/ansible/latest/modules/service_module.html)
