# 🚀 Full CI/CD DevOps Workflow Using Minikube (No Prebuilt Images)

## 🔁 CI/CD Flow

**GitHub → Jenkins → Ansible → Docker (inside Minikube) → Kubernetes Deployment**

---

## 📦 Project Goal

Automate the entire application lifecycle — from code push to production — **without using prebuilt Docker or Jenkins images**, on AWS EC2, with Kubernetes managed via Minikube.

---

## 🧱 Infrastructure Overview

| EC2 Instance     | Role            | Tools Installed                      |
| ---------------- | --------------- | ------------------------------------ |
| `jenkins-server` | CI Controller   | Jenkins, Git, OpenJDK                |
| `ansible-server` | CD Orchestrator | Ansible, SSH, Git, Python            |
| `minikube-node`  | Target Node     | Docker, Minikube, kubectl, Flask App |

✅ **Ports to open in EC2 security group**:

* `22` for SSH
* `8080` for Jenkins
* `30000-32767` for NodePort services

---

## ✅ Step-by-Step Process

### 1️⃣ Set up Minikube Node

```bash
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker $USER && newgrp docker

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

minikube start --driver=docker
```

---

### 2️⃣ Set up Jenkins Server

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk wget git
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /etc/apt/trusted.gpg.d/jenkins.asc > /dev/null
echo "deb https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt update
sudo apt install -y jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

Then open Jenkins in the browser:
`http://<jenkins-ec2-ip>:8080`

---

### 3️⃣ Set up Ansible Server

```bash
sudo apt update
sudo apt install -y python3 python3-pip ansible git sshpass
mkdir -p ~/ci-cd-pipeline/{ansible,flask-app}
```

---

### 4️⃣ Create Flask App (`flask-app/`)

* `app.py`: Simple Flask web server
* `Dockerfile`: Manual Docker image
* `requirements.txt`: Add `flask`
* `k8s.yaml`: Deployment + NodePort service

---

### 5️⃣ Create Ansible Files (`ansible/`)

📄 `inventory.ini`:

```ini
[minikube]
<minikube-ip> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/key.pem
```

📄 `deploy.yaml`:

```yaml
- name: Deploy Flask App to Minikube
  hosts: minikube
  become: yes
  tasks:
    - name: Set Minikube Docker Environment
      shell: eval $(minikube docker-env)
      args:
        executable: /bin/bash

    - name: Copy application code
      copy:
        src: ../flask-app/
        dest: /home/ubuntu/flask-app/

    - name: Build Docker Image
      shell: |
        eval $(minikube docker-env)
        cd /home/ubuntu/flask-app
        docker build -t flask-app:latest .

    - name: Deploy app using Kubernetes
      shell: |
        cd /home/ubuntu/flask-app
        kubectl apply -f k8s.yaml
```

---

### 6️⃣ Jenkins Freestyle Project

* Source Code Management → Git
* Build Step → Execute shell:

```bash
cd ~/ci-cd-pipeline/ansible
ansible-playbook -i inventory.ini deploy.yaml
```

---

### 7️⃣ Access Your Application

```bash
kubectl get svc
```

Then open:

```
http://<minikube-ec2-ip>:<node-port>
```

---

## 📁 Folder Structure

```
ci-cd-pipeline/
├── ansible/
│   ├── deploy.yaml
│   └── inventory.ini ✅
├── flask-app/
│   ├── app.py
│   ├── Dockerfile
│   ├── requirements.txt
│   └── k8s.yaml
```

---

## 🐞 Problems Faced & Solutions

| Problem                           | Fix                                                           |
| --------------------------------- | ------------------------------------------------------------- |
| `minikube docker-env` not working | Must run `eval $(minikube docker-env)` in shell               |
| Docker image not found in K8s     | Use `imagePullPolicy: Never` and build inside Minikube        |
| SSH permission denied (Ansible)   | Check `.pem` permissions and known\_hosts file                |
| `kubectl` failed in playbook      | Ensure Minikube is active and `docker-env` is sourced         |
| `ansible-playbook` not found      | Add full path or install Ansible for Jenkins user             |
| Host key verification failed      | Pre-authorize SSH in Jenkins using `StrictHostKeyChecking=no` |
| K8s Pod in `ErrImagePull`         | Rebuild Docker image inside Minikube                          |

---

## 🧼 Cleanup

```bash
kubectl delete -f k8s.yaml
docker rmi flask-app:latest
```

---

## ✅ Summary

| Stage         | Tool       | Notes                                           |
| ------------- | ---------- | ----------------------------------------------- |
| Code Repo     | GitHub     | Triggered via Jenkins webhook                   |
| CI Build      | Jenkins    | SSH to Ansible, pulls latest code               |
| CD Deploy     | Ansible    | Deploys app on Minikube                         |
| Image Build   | Docker     | Built manually inside Minikube                  |
| Orchestration | Kubernetes | Exposes app via NodePort                        |
| Hosting       | EC2        | App is reachable at `<minikube-ip>:<node-port>` |

