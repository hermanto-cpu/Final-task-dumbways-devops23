# 📘 Final Task

## 1 - Provisioning

**Instructions**

- Attach SSH keys & IP configuration to all VMs
- Server Configuration using Ansible

---

1. Rebuild semua server lalu gunakan 1 SSH key ke kedua server tersebut
   ![ansible](img/ssh.png)
2. Gunakan 1 key SSH lalu konfigurasikan ke semua seperti di Biznetgio, Github, Jenkins, Local windows dan WSL (untuk Ansible). Buat juga config di local untuk mempermudah masuk ke server via SSH.

```
Host vps-app
    HostName 103.127.137.206
    User totywan
    IdentityFile ~/.ssh/key

Host gateway
    HostName 103.127.138.159
    User totywan
    IdentityFile ~/.ssh/key
```

3. Apabila SSH key yang dipakai pernah digunakan maka lakukan seperti gambar dibawah:
   ![ansible](img/key.png)

4. Login ulang, jika sudah selesai.
5. Buat Inventory untuk konfigurasi server menggunakan ansible

```bash
[app_server]
103.127.137.206

[gateway_server]
103.127.138.159

[app_server:vars]
ansible_user=totywan
ansible_ssh_private_key_file=~/.ssh/key
ansible_python_interpreter=/usr/bin/python3.10

[gateway_server:vars]
ansible_user=totywan
ansible_ssh_private_key_file=~/.ssh/key
ansible_python_interpreter=/usr/bin/python3.10
```

6.  Buat file ansible.cfg untuk mengkonfigurasi behavior Ansible saat kita menjalankan perintah-perintah ansible

```bash
[defaults]
inventory=./inventory
host_key_checking=false
```

    - host_key_checking digunakan untuk menghindari host key checking saat pertama kali connect ke server baru

---

## 2 - Repository

**Instructions**

- Create a repository on Github or Gitlab
- **Private** repository access
- Set up 2 branches
  - Staging
  - Production
- Each Branch have their own CI/CD

---

1. Buat repository untuk aplikasi di github dan set private

2. Fork repo fe-dumbmerch dan be-dumbmerch kemudian pindahkan ke repo github yang sudah dibuat

3. Buat branch staging dan branch production kemudian push repository
   ![ansible](img/1.png)
   ![ansible](img/2.png)
   ![ansible](img/3.png)

4. Karena menggunakan fork dan hanya mengcopy ke repo kita, maka pada git akan muncul `mode 1600000` yang berarti direktori fe dan be merupakan submodule Git. Hal ini dapat membuat CI/CD dan Docker Build tidak aman karena:

- CI/CD bisa gagal kalau tidak diatur untuk clone submodule (git clone --recurse-submodules)

- Docker build bisa error, terutama jika Dockerfile butuh isi folder tetapi yang ada cuma placeholder submodule

Oleh karena itu lakukan

```
# Hapus submodule dari index
git rm --cached fe-dumbmerch
git rm --cached be-dumbmerch

# Hapus .git folder di dalamnya
rm -rf fe-dumbmerch/.git
rm -rf be-dumbmerch/.git

# Tambahkan ulang sebagai folder biasa
git add .
git commit -m "Fix: convert submodules to normal folders"
git push origin staging

```

Lakukan hal yang sama di branch production

5. Buat Jenkinsfile di tiap branch untuk CI/CD tiap branch
   ![ansible](img/4.png)
   ![ansible](img/5.png)

## 3 - Servers

**Instructions**

- Create new user `finaltask-$USER` (this new user was your final tasks playground)
- Server login with SSH key and Password
- Create a working **SSH config** to log into servers
- Only use **1 SSH keys** for all purpose (Repository, CI/CD etc.)
- UFW enabled with only used ports allowed
- Change ssh port from (22) to (1234)

---

1. Buat file `.yml` yang berisi task SSH config yang akan dijalankan via ansible-playbook

```.yml
- hosts: app_server, gateway_server
  become: true
  tasks:
    - name: Create user finaltask-totywan
      user:
        name: finaltask-totywan
        shell: /bin/bash
        groups: sudo
        append: yes
        create_home: yes

    - name: Set authorized key for finaltask-totywan
      authorized_key:
        user: finaltask-totywan
        state: present
        key: "{{ lookup('file', '~/.ssh/key.pub') }}"

    - name: Set password login untuk finaltask-totywan
      user:
        name: finaltask-totywan
        password: "{{ 'totywan' | password_hash('sha512') }}"

    - name: Change SSH port to 1234
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?Port'
        line: 'Port 1234'
        state: present

    - name: Allow ports used via UFW
      ufw:
        rule: allow
        port: "{{ item }}"
      loop:
        - 1234   # SSH
        - 80     # HTTP
        - 443    # HTTPS
        - 5432   # PostgreSQL
        - 9000   # Sonarqube
        - 5000

    - name: Enable UFW
      ufw:
        state: enabled
        policy: deny

    - name: Restart SSH to apply port change
      service:
        name: ssh
        state: restarted
    - name: Allow finaltask-totywan to sudo without password
      lineinfile:
        dest: /etc/sudoers.d/finaltask-totywan
        line: "finaltask-totywan ALL=(ALL) NOPASSWD:ALL"
        create: yes
        state: present
        mode: '0440'
```

2.  Karena telah membuat user baru yang digunakan untuk keseluruhan final task, ubah inventory dari Ansible dan config ssh untuk user yang mengelola VM

```inventory
[app_server]
103.127.137.206

[gateway_server]
103.127.138.159

[app_server:vars]
ansible_user=finaltask-totywan
ansible_ssh_private_key_file=~/.ssh/key
ansible_python_interpreter=/usr/bin/python3.10
ansible_port=1234

[gateway_server:vars]
ansible_user=finaltask-totywan
ansible_ssh_private_key_file=~/.ssh/key
ansible_python_interpreter=/usr/bin/python3.10
ansible_port=1234
```

```config
Host vps-app
    HostName 103.127.137.206
    User finaltask-totywan
    Port 1234
    IdentityFile ~/.ssh/key

Host gateway
    HostName 103.127.138.159
    User finaltask-totywan
    Port 1234
    IdentityFile ~/.ssh/key
```

## 4 - Container Registry

**Instructions**

[ *Docker Registry* ]

- Deploy Docker Registry Private on this server
- Push your image into Your Own Docker Registry
- reverse proxy for docker registry was registry.{your_name}.studentdumbways.my.id

[*Reference*]
([Docker Registry Private](https://hub.docker.com/_/registry))

---

## 5 - Deployment

**_Instruction_**

[ *Database* ]

- App database using _PostgreSQL_
- Deploy postgresql on top docker
- Set the volume location in `/home/$USER/`
- Allow database to remote from another server\*

[ *Application* ]

- Create a Docker image for frontend & backend
- Staging & Production: A lightweight docker image (as small as possible)
- Building Docker image on every environment using docker multistage build\*
- Create load balancing for frontend and backend\*

---

## 6 - CI/CD

**Instructions**

[ *CI/CD* ]

- Create a pipeline running:
  - Repository pull
  - Image build
  - Testing Your Code
  - Push Image into your own docker registry private
  - SSH into your server
  - Pull image from docker registry private
  - Redeploy your deployment apps
- For testing Stage, you must use sonarqube for testing your code quality
  - Sonarqube\*
    - On your cicd workflow, use sonarqube for testing your quality code.
    - You can use others testing tools like trivy for scanning your images or some.
    - And try test your running app by using `wget spider`

---

## 7 - Monitoring

**Instructions**

- Create Basic Auth into your Prometheus\*
- Monitor resources for all your servers
- Create a fully working dashboard in Grafana
  - Disk
  - Memory Usage
  - CPU Usage
  - VM Network
  - Monitoring all of container resources on VM
- Grafana Alert/Prometheus Alertmanager for:
  - Send Notification to Telegram
  - CPU Usage
  - RAM Usage
  - Free Storage
  - Network I/O (NGINX Monitoring)

---

## 8 - Web Server

**Instructions**

- All domains are HTTPS
- Create Bash Script for Automatic renewal for Certificates
- Create domains:

[ *Monitoring* ]

- exporter.<name>.studentdumbways.my.id - Node Exporter
- prom.<name>.studentdumbways.my.id - Prometheus
- monitoring.<name>.studentumbways.my.id - Grafana

[ *Docker Registry* ]

- registry.<name>.studentdumbways.my.id - Docker Registry

[ *Staging* ]

- staging.<name>.studentdumbways.my.id - App
- api.staging.<name>.studentdumbways.my.id - Backend API

[ *Production* ]

- <name>.studentdumbways.my.id - App
- api.<name>.studentdumbways.my.id - Backend API

---
