a **complete step-by-step guide** to connect an **EC2 instance as a Jenkins agent** and create a pipeline that runs jobs on it.

---

## 🧩 **Section 1: Prerequisites**

Before starting, ensure:

* Jenkins master (controller) is already installed and accessible (`http://<jenkins-server>:8080`)
* You have **one EC2 instance** running as **Jenkins Controller**
* You have **another EC2 instance** that will act as the **Jenkins Agent**
* Both instances:

  * Run Linux (Ubuntu or Amazon Linux)
  * Can communicate over **port 22 (SSH)** and **port 8080**
  * Have Java installed (`java -version`)

---

## ⚙️ **Section 2: Configure EC2 Agent Instance**

### 1. Connect to Agent EC2

```bash
ssh -i your-key.pem ec2-user@<agent-public-ip>
```

### 2. Install Java

```bash
sudo yum install java-11-openjdk -y      # Amazon Linux
# OR
sudo apt install openjdk-11-jre -y       # Ubuntu
```

### 3. Create Jenkins User

```bash
sudo useradd jenkins
sudo passwd jenkins
sudo usermod -aG sudo jenkins
```

### 4. Generate SSH Key (optional, for mutual auth)

```bash
sudo su - jenkins
ssh-keygen -t rsa -b 4096
cat ~/.ssh/id_rsa.pub
```

Copy this public key and add it to the Jenkins Controller’s `~/.ssh/authorized_keys` if you plan to use SSH connection.

---

## 🖥️ **Section 3: Configure Jenkins Master (Controller)**

### 1. Add the EC2 Agent Node

* Open Jenkins → **Manage Jenkins → Nodes → New Node**
* Name: `EC2-Agent`
* Select **Permanent Agent**
* Click **OK**

### 2. Configure Node

* **Remote root directory:** `/home/jenkins`
* **Labels:** `ec2-agent`
* **Usage:** “Use this node as much as possible”

### 3. Launch Method Options

#### Option A: **SSH Launch**

If Jenkins controller can SSH into the EC2 agent:

* Launch method: **Launch agents via SSH**
* Enter:

  * Host: `<agent-private-ip>`
  * Credentials: Add → SSH Username with private key
  * Username: `ec2-user` or `jenkins`
  * Private key: paste PEM key content
* Save and click **Launch agent**

#### Option B: **JNLP (Inbound)**

If agent connects to master:

* Launch method: **Launch agent via Java Web Start (JNLP)**
* Save node configuration
* On Agent EC2:

  ```bash
  mkdir ~/jenkins-agent && cd ~/jenkins-agent
  wget http://<jenkins-master-ip>:8080/jnlpJars/agent.jar
  java -jar agent.jar -jnlpUrl http://<jenkins-master-ip>:8080/computer/EC2-Agent/slave-agent.jnlp -secret <SECRET> -workDir "/home/jenkins"
  ```
* The agent should show **Connected** in Jenkins UI.

---

## 🧪 **Section 4: Verify the Agent Connection**

In Jenkins:

* Go to **Manage Jenkins → Nodes**
* The `EC2-Agent` should show **Connected** and **Online**

Run a simple test job:

```bash
echo "Hello from EC2 Agent"
hostname
```

---

## 🧰 **Section 5: Create a Pipeline to Use EC2 Agent**

### 1. Create New Pipeline

* Go to **Jenkins → New Item → Pipeline**
* Name: `EC2-Agent-Pipeline`
* Click **OK**

### 2. Add Pipeline Script

```groovy
pipeline {
    agent { label 'ec2-agent' }

    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning repository...'
                git branch: 'main', url: 'https://github.com/atulkamble/pythonhelloworld.git'
            }
        }

        stage('Build') {
            steps {
                echo 'Building application...'
                sh 'python3 --version'
                sh 'echo "Build Successful!"'
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'echo "Tests completed successfully!"'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying to environment...'
                sh 'echo "Deployment done!"'
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
        }
    }
}
```

### 3. Save and Run

* Click **Build Now**
* Watch console logs — you’ll see it runs on **EC2-Agent**:

  ```
  Running on EC2-Agent in /home/jenkins/workspace/EC2-Agent-Pipeline
  ```

---

## 🧩 **Section 6: Validation & Maintenance**

### Validate Agent:

```bash
hostname
java -version
ps -ef | grep agent.jar
```

### Auto-start agent after reboot:

Add in `/etc/rc.local` or `systemd` unit:

```bash
[Unit]
Description=Jenkins Agent
After=network.target

[Service]
User=jenkins
ExecStart=/usr/bin/java -jar /home/jenkins/jenkins-agent/agent.jar -jnlpUrl http://<jenkins-master>:8080/computer/EC2-Agent/slave-agent.jnlp -secret <SECRET> -workDir "/home/jenkins"
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable jenkins-agent
sudo systemctl start jenkins-agent
```

---

## ✅ **Summary Checklist**

| Step | Task                                    | Done |
| ---- | --------------------------------------- | ---- |
| 1    | EC2 Agent prepared with Java & user     | ✅    |
| 2    | Jenkins node added                      | ✅    |
| 3    | Connection verified (SSH/JNLP)          | ✅    |
| 4    | Pipeline created with label `ec2-agent` | ✅    |
| 5    | Job executed successfully on EC2        | ✅    |

---
