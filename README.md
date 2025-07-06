#  CI/CD Pipeline: Git + Ansible + Tomcat

This project implements a **CI/CD pipeline** to deploy a Java `.war` file on a **Tomcat** server using **Git hooks** and **Ansible**.

When a developer pushes code to the Git server, a `post-receive` hook:
1. Pulls the latest source code.
2. Change code and builds the project with Maven.
3. Push code to git server
4. Triggers an Ansible playbook on a remote Ansible server.
5. Ansible deploys the updated `.war` file to the Tomcat server.

---

## 🧱 Servers Architecture

```text
┌────────────┐        git push       
│ Developer  │ ──────────────────────────── |
└────────────┘                              |
                                            ▼
                                    ┌─────────────┐
                                    │ Git Server  │
                                    │ 10.10.10.20 │
                                    └──────┬──────┘
                                           │ post-receive hook
                                           ▼
                                    ┌────────────────┐
                                    │ Ansible Server │
                                    │ 10.10.10.17    │
                                    └──────┬─────────┘
                                           │ ansible-playbook
                                           ▼    
                                    ┌──────────────┐
                                    │ Tomcat Server│
                                    │ 10.10.10.14  │
                                    └──────────────┘
```



## 🧱 Repo Architecture
```txt
.
├── ansible
│   ├── cicd.yml
│   ├── inventory
│   │   └── cicd
│   └── roles
│       └── cicd
│           ├── defaults
│           │   └── main.yml
│           ├── files
│              └── my-webapp-1.0-SNAPSHOT.war
│           
│               
└── git
    ├── app
    │   ├── pom.xml
    │   ├── src
    │   │   └── main
    │   │       ├── java
    │   │       │   └── com
    │   │       │       └── example
    │   │       │           └── App.java
    │   └── target
    │       ├── classes
    │       │       └── web.xml
    │       └── my-webapp-1.0-SNAPSHOT.war
    └── hooks
        └── post-receive
```


## ⚙️ Step-by-Step Setup

### 1. Setup Bare Git Repo (on Git Server `10.10.10.20`)

```bash
cd /srv/git
git init --bare my-java-app.git
```

### 2. Add post-receive Hook
Create the post-receive file at:

```bash
/srv/git/my-java-app.git/hooks/post-receive
```
Content:
```bash
#!/bin/bash

REPO_DIR="/srv/git/my-java-app.git"
WORK_DIR="/home/my-java-app"
ANSIBLE_SERVER="10.10.10.17"

echo "===> Pull latest code..."
cd "$WORK_DIR" || exit 1
git pull origin master

echo "===> Building project..."
mvn clean package

echo "===> Running Ansible playbook on $ANSIBLE_SERVER ..."
ssh root@$ANSIBLE_SERVER "cd /home/ansible/provision && ansible-playbook -i inventory/cicd cicd.yml"
```

Make it executable:
```bash
chmod +x /srv/git/my-java-app.git/hooks/post-receive
```

### 3. Clone Repo in Work Directory (on Git Server)
```bash
cd /home
git clone /srv/git/my-java-app.git my-java-app
```
### 4. Configure Ansible Playbook (on Ansible Server 10.10.10.17)
Edit inventory file:
```ini
# inventory/cicd
[servers]
10.10.10.20
10.10.10.14
localhost
```

Key task logic (inside roles/cicd/tasks/main.yml):

- Fetch WAR file from Git server

- Copy WAR to Tomcat server

- Remove previous deployments

### 5. Setup Tomcat Server (on 10.10.10.14)
You can follow this GitHub repository to install and configure Apache Tomcat on Ubuntu:

👉 [Tomcat Ubuntu Setup by mehdi-khaksari](https://github.com/mehdi-khaksari/tomcat-ubuntu-setup)

Tomcat WAR Deployment Path:
```bash
/opt/tomcat/webapps/
```
When .war is copied here, Tomcat auto-deploys the app:

- /opt/tomcat/webapps/my-webapp-1.0-SNAPSHOT.war

- Tomcat extracts it to:
```swift
/opt/tomcat/webapps/my-webapp-1.0-SNAPSHOT/
```
### 6. Trigger the Pipeline
Once you push new code to the Git server (e.g. via developer machine):
```bash
git remote add origin ssh://git@10.10.10.20/srv/git/my-java-app.git
git push origin master
```
This will:

- Pull the latest code
- Build the .war file using Maven
- Run the Ansible playbook
- Deploy to Tomcat

### ✅Result

- You’ll have continuous deployment working:
- Developers push to Git
- Hook triggers build
- Ansible deploys the WAR to Tomcat

### 🔐 Security Tips
- Limit SSH access between Git/Ansible servers.
- Use SSH keys between Git → Ansible.
- Ensure Tomcat runs as a non-root user.

# 📬 Contact
Feel free to fork, improve, and contribute!

Author: mehdi khaksari 

Email: mahdikhaksari36@gmail.com
