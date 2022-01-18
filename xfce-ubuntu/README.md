Ansible role for setting XFCE GUI For Ubuntu 20.04

![Ubuntu](https://img.shields.io/badge/-ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![XFCE](https://img.shields.io/badge/-xfce-2284F2?style=for-the-badge&logo=xfce&logoColor=white)

- It's expected to pass the ubuntu password via the variable `ubuntu_password` so the full command to execute the role can be 
```bash
ansible-playbook xfce-ubuntu.yaml --extra-vars ubuntu_password=<password>
```
- The role was made for AWS Ubuntu Instance, the security Group should enable RDP Port number **3389**
- After the role is executed, you should be able to reach the instance using a remote desktop client tool like [`Remmina`](https://remmina.org/)