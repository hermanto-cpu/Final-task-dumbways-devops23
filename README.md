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

3. Coba masuk menggunakan user tersebut dan periksa UFW statusnya
   ![ansible](img/final-user.png)

## 4 - Container Registry

**Instructions**

[ *Docker Registry* ]

- Deploy Docker Registry Private on this server
- Push your image into Your Own Docker Registry
- reverse proxy for docker registry was registry.{your_name}.studentdumbways.my.id

[*Reference*]
([Docker Registry Private](https://hub.docker.com/_/registry))

---

Arsitektur

```

┌──────────────┐     registry.hermanto.studentdumbways.my.id
│ Gateway VPS  │  ⇐═════════════════════════════════════  (HTTPS Reverse Proxy)
│ (nginx)      │
└──────────────┘           │
                           ▼
                 ┌────────────────┐
                 │ App Server     │
                 │ Docker Registry│
                 └────────────────┘

```

1. Buat playbook Install Docker Registry di app server

```.yml
- name: Install Docker
  hosts: app_server
  become: true
  tasks:
    - name: Install required packages
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present
        update_cache: yes

    - name: Add Docker GPG key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker APT repository
      apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present

    - name: Install Docker
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
        state: latest

    - name: Enable and start Docker service
      systemd:
        name: docker
        enabled: yes
        state: started
    - name: Add 'finaltask-totywan' user to docker group
      user:
        name: finaltask-totywan
        groups: docker
        append: yes

    - name: Create registry data dir
      file:
        path: /home/finaltask-totywan/registry/data
        state: directory
        recurse: yes
        owner: finaltask-totywan

    - name: Run Docker Registry container
      docker_container:
        name: registry
        image: registry:2
        state: started
        restart_policy: always
        published_ports:
          - "5000:5000"
        volumes:
          - /home/finaltask-totywan/registry/data:/var/lib/registry
```

2. Cek apakah registry berhasil berjalan dengan baik.
   ![ansible](img/registry.png)

3. Buat playbook install nginx di gateway server

```.yml
- name: Install and configure Nginx as reverse proxy for Docker Registry
  hosts: gateway_server
  become: true
  vars:
    registry_domain: "registry.hermanto.studentdumbways.my.id"
    app_server_ip: "103.127.137.206"
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure Nginx is started and enabled
      systemd:
        name: nginx
        state: started
        enabled: true

    - name: Configure Nginx reverse proxy for Docker Registry
      copy:
        dest: /etc/nginx/sites-available/docker-registry
        content: |
          server {
              listen 80;
              server_name {{ registry_domain }};

              location / {
                  proxy_pass http://{{ app_server_ip }}:5000;
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                  proxy_set_header X-Forwarded-Proto $scheme;
              }
          }
      notify: Reload Nginx

    - name: Enable Docker Registry site
      file:
        src: /etc/nginx/sites-available/docker-registry
        dest: /etc/nginx/sites-enabled/docker-registry
        state: link
        force: true

    - name: Remove default nginx site
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent

  handlers:
    - name: Reload Nginx
      service:
        name: nginx
        state: reloaded
```

4. Tes apakah nginx sudah running dan reverse proxy sudah benar. (sementara menggunakan HTTP karena SSL akan dikonfigurasi di akhir menggunakan wildcard)
   ![ansible](img/r1.png)
   ![ansible](img/r2.png)
   ![ansible](img/r3.png)

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

1. Konfigurasi SSH dengan github terlebih dahulu, agar dapat melakukan pull dari repo app, dapat menggunakan Ansible dengan playbook di bawah:

```ansible
- name: Setup SSH for GitHub
  hosts: app_server, gateway_server
  become: true
  vars:
    ssh_dir: "/home/finaltask-totywan/.ssh"
    github_repo: "git@github.com:hermanto-cpu/final-task-app-dumbways.git"
    repo_dir: "/home/finaltask-totywan/final-task-app"

  tasks:
    - name: Ensure .ssh directory exists
      file:
        path: "{{ ssh_dir }}"
        state: directory
        owner: finaltask-totywan
        group: finaltask-totywan
        mode: "0700"

    - name: Copy private key
      copy:
        src: ~/.ssh/key
        dest: "{{ ssh_dir }}/key"
        owner: finaltask-totywan
        group: finaltask-totywan
        mode: "0600"

    - name: Copy public key
      copy:
        src: ~/.ssh/key.pub
        dest: "{{ ssh_dir }}/key.pub"
        owner: finaltask-totywan
        group: finaltask-totywan
        mode: "0644"

    - name: Configure SSH to use custom key for github.com
      copy:
        dest: "{{ ssh_dir }}/config"
        content: |
          Host github.com
            HostName github.com
            User git
            IdentityFile {{ ssh_dir }}/key
            StrictHostKeyChecking no
        owner: finaltask-totywan
        group: finaltask-totywan
        mode: "0644"

    - name: Ensure git is installed
      apt:
        name: git
        state: present
        update_cache: yes

- name: Clone Repo to app server
  hosts: app_server
  become: true
  tasks:
    - name: Clone GitHub repo (final-task-app-dumbways)
      become_user: finaltask-totywan
      git:
        repo: "{{ github_repo }}"
        dest: "{{ repo_dir }}"
        version: staging
        accept_hostkey: yes
```

2. Cek apakah github dapat terkoneksi melalui SSH dan repo terclone
   ![ansible](img/ssh-git.png)

3. Buat `.env` file di fe-dumbmerch

```
cd /home/finaltask-totywan/final-task-app/fe-dumbmerch
echo "REACT_APP_BASEURL=https://api.hermanto.studentdumbways.my.id/api/v1" > .env
```

4. Buat Dockerfile didalam fe-dumbmerch

```
# Stage 1: Build React App
FROM node:16-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Serve with NGINX
FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

- Karena fe-dumbmerch adalah React app (Single Page Application) yang saat di build (npm run build) akan menghasilkan static files di /build

- Static files (HTML, JS, CSS) tersebut idealnya disajikan dengan web server seperti NGINX

- Oleh karena itu, di Dockerfile yang kita lakukan adalah Membangun project di NodeJS → Copy hasilnya ke image NGINX → Serve static file menggunakan NGINX

- command `-g "daemon off;"`: memberi tahu nginx jangan masuk background (daemon), tetap di foreground. Karena Docker container akan berhenti kalau proses utamanya selesai. Maka nginx harus tetap aktif di depan (foreground).

- Images hasil Dockerfile tersebut akan lebih ringan.

5. Build docker file menggunakan nama domain registry beserta tag agar dapat langsung kita pull ke registry private kita. Ex: `docker build -t registry.hermanto.studentdumbways.my.id/fe-dumbmerch:staging .`
   ![ansible](img/build.png)

6. Cek Docker Images dan jalankan image front-end yang telah dibuild `docker run -d --name frontend-app -p 3000:80 registry.hermanto.studentdumbways.my.id/fe-dumbmerch:staging`
   ![ansible](img/cek.png)

7. Cek apakah front-end dapat diakses dan dirender melalui browser
   ![ansible](img/fe.png)
   ![ansible](img/fe2.png)

8. Push Image front end ke docker registry

- Karena domain untuk reverse proxy registry saya belum memiliki SSL maka harus dilakukan konfigurasi terlebih dahulu agar Docker dapat melakukan push melalui HTTP untuk domain tersebut.
- Edit file Docker daemon config di app server`sudo nano /etc/docker/daemon.json`
- Masukkan konfigurasi

```{
  "insecure-registries": ["registry.hermanto.studentdumbways.my.id"]
  }

```

- Restart Docker daemon:

```

sudo systemctl daemon-reexec
sudo systemctl restart docker
```

![ansible](img/push.png)

9. Sekarang kita lanjut untuk deployment backend dan database, Buat docker compose untuk database

10. Sesuaikan .env untuk backend
    ![ansible](img/env.png)
    ![ansible](img/env-be.png)

11. Buat Dockerfile untuk backend

```Dockerfile
# Stage 1 - Build
FROM golang:1.16 AS builder

WORKDIR /app

COPY go.mod ./
COPY go.sum ./
RUN go mod download

COPY . .
RUN go build -o app .

# Stage 2 - Run
FROM debian:bullseye-slim

WORKDIR /app

COPY --from=builder /app/app .
COPY --from=builder /app/.env .

EXPOSE 5001

CMD ["./app"]

```

12. Build Dockerfile `docker build -t registry.hermanto.studentdumbways.my.id/be-dumbmerch:staging .`
13. Buat docker compose agar database dan backend dapat satu network container

```docker-compose.yml
version: '3.8'
services:
  db:
    image: postgres:13
    container_name: postgres-dumbmerch
    restart: always
    environment:
      POSTGRES_USER: dumbuser
      POSTGRES_PASSWORD: dumbpass
      POSTGRES_DB: dumbmerch
    ports:
      - "5432:5432"
    volumes:
      - /home/finaltask-totywan/postgres-data:/var/lib/postgresql/data

  backend:
    build:
      context: ./be-dumbmerch
    container_name: backend-staging
    env_file:
      - ./be-dumbmerch/.env
    ports:
      - "5001:5001"
    depends_on:
      - db

```

14. Jalankan `docker compose up -d`

15. Cek logs dan juga coba akses via browser
    ![ansible](img/logs.png)
    ![ansible](img/be.png)

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

```

```
