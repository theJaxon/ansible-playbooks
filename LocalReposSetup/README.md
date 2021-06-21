### Local Repository setup using ansible:
![RHEL8](https://img.shields.io/badge/-RHEL8-EE0000?style=for-the-badge&logo=red%20hat&logoColor=white)
- From the article [setting up local repos using ansible](https://www.linkedin.com/pulse/setting-up-local-repos-using-ansible-mohamed-mostafa)

When working with ansible you might find that you install lots of software packages and because of this you may end up consuming your monthly internet quota, the solution is to disable all the repos that come enabled with your distribution and start using local repos to do the job, the steps are simple and after setting up the repos your software will be installed in no time.

- This is intended for either CentOS 8 or RHEL 8
- This method consumes lots of space so make sure you've got sufficient disk space
- This is intended for practice only, usually you’ll want your software to stay up to date and so you’ll avoid using this method.

First extract the **AppStream** and **BaseOS** directories from CentOS downloaded ISO 

Place the folders in the directory where your `Vagrantfile` exists, if the machines are already up then make sure to run `vagrant rsync-auto` so that the files can be located at /vagrant/ directory

Write an ansible playbook that runs on all the managed nodes to do the following:

- Disable or remove all existing repos
- Create AppStream and BaseOS local repos

* In case of just disabling the repos, move the `CentOS-AppStream.repo` and `CentOS-BaseStream.repo` outside of `/etc/yum.repos.d` to avoid “Repository is listed more than once in the configuration” error

```yaml
---
- hosts: all
  vars:
    local_repos:
      - file: AppStream
        name: AppStream
        description: AppStream local repo
        baseurl: file:///vagrant/AppStream

      - file: BaseOS
        name: BaseOS
        description: BaseOS local repo
        baseurl: file:///vagrant/BaseOS

  tasks:
    - name: remove all repos existing in /etc/yum.repos.d/
      shell: "rm -rf /etc/yum.repos.d/*"

    #- name: disable all repos insted of removing them
    #  command: "yum-config-manager --disable \* "

    - name: setup BaseOS and AppStream local repos
      yum_repository:
        file: "{{ item.file }}"
        name: "{{ item.name }}"
        description: "{{ item.description }}"
        baseurl: "{{ item.baseurl }}"
        gpgcheck: False
      loop: "{{ local_repos }}"
```

Now run the command `yum repolist` on any of the hosts

You should be able to see the configured repos with their corresponding description

Finally run `yum list` command to verify that all packages from both repos can be listed.