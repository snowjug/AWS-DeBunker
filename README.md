End-to-End Manual CI/CD Guide (Jenkins + SonarQube + Docker on AWS EC2)
This README-style guide walks through every click,, and setting required to spin up three Amazon Linux 2 t2.medium instances—one each for Jenkins, SonarQube, and Docker—and then glue them together manually through their browser interfaces. It covers instance provisioning, plugin installation (SonarQube Scanner, SSH2 Easy), GitHub webhooks, token generation, job creation, and all the tricky UI fields for project properties so you can reproduce the workflow for the DeBunker repository.

EC2 Foundation
Designing the Instance Set
Instance Name	Purpose	Minimum AWS Spec	Open Inbound Ports
jenkins-server	CI orchestration	t2.medium, 30 GiB gp3	22, 8080
sonarqube-server	Static code analysis	t2.medium, 50 GiB gp3 (IO-heavy)	22, 9000
docker-server	Image build & runtime	t2.medium, 30 GiB gp3	22, 80/443 (app)
Security Rule Golden Rule: never leave “All Traffic 0.0.0.0/0” enabled once testing completes. Restrict each port to specific source CIDRs—especially SSH.

Launching the Instances
Console → EC2 → Launch Instance.

Select Amazon Linux 2 AMI (x86_64).

Choose t2.medium.

Add 30 GiB (Jenkins/Docker) or 50 GiB (SonarQube) gp3 volume.

Tag the instance (Name=jenkins-server etc.).

Create a security group per instance using the ports above.

Download a key pair (PEM) for SSH.

After status checks pass, note each public IP (or allocate Elastic IP for static DNS).

Jenkins Server: Complete UI Setup
1 – Install Prerequisites
bash
sudo yum update -y
sudo yum install java-17-amazon-corretto-devel -y     # Full JDK[1]
2 – Add Jenkins Repository & Install
bash
sudo wget -O /etc/yum.repos.d/jenkins.repo <https://pkg.jenkins.io/redhat-stable/jenkins.repo>[6]
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install jenkins -y
sudo systemctl enable --now jenkins
3 – Unlock & Initial Wizard
Browser → http://<JENKINS_IP>:8080.

Copy unlock key:

bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Select Install suggested plugins.

Create the admin user.

4 – Install Needed Plugins Manually
Navigate: Manage Jenkins → Manage Plugins → Available.

Plugin	Purpose	Notes
SonarQube Scanner	Adds Sonar “Execute Scanner” build step & system section	Install, restart Jenkins
GitHub plugin	Hyperlinks + webhook trigger	
SSH2 Easy	SSH command/SFTP build steps	Shows “Server Groups Center” in Configure System
Verify under Installed tab after restart.

5 – Add Global Jenkins Credentials
Manage Jenkins → Credentials → System → Global → Add Credentials.

Kind: “SSH Username with private key”
ID: github_ssh – paste your private key that matches the deploy key on GitHub.

Kind: “Secret text”
ID: sonar_token – paste the SonarQube user/project token generated later.

SonarQube Server Setup
1 – Java & Download
bash
sudo yum install java-17-amazon-corretto-devel unzip -y
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.5.1.90531.zip
unzip sonarqube-10.5.1.90531.zip
sudo mv sonarqube-10.5.1.90531 /opt/sonarqube
sudo useradd sonarqube
sudo chown -R sonarqube:sonarqube /opt/sonarqube
2 – Run as Service
bash
sudo tee /etc/systemd/system/sonarqube.service <<'EOF'
[Unit]
Description=SonarQube
After=network.target

[Service]
Type=forking
User=sonarqube
Group=sonarqube
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
LimitNOFILE=65536
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now sonarqube
3 – First-Run Browser Steps
Visit http://<SONAR_IP>:9000.

Log in admin / admin → change password.

User → My Account → Security → Generate Token (jenkins-scan) and copy it.

Create Project → Manually

Project key: debunker

Display name: DeBunker

SonarQube now awaits incoming scans.

Docker Host Preparation
bash
sudo yum update -y
sudo amazon-linux-extras install docker
sudo service docker start
sudo usermod -aG docker ec2-user
logout && login      # activate group
docker --version     # confirm[3]
Add port 80 (or your app port) in the Docker server’s security group to world or VPC as required.

Jenkins Global Configuration for SonarQube & SSH2 Easy
Manage Jenkins → Configure System.

SonarQube Servers section (created by plugin):

Name: sonar-server

Server URL: http://<SONAR_IP>:9000

Credentials: select sonar_token (Secret Text).

Check Environment variables box to inject $SONAR_HOST_URL, etc.

SonarScanner installations (still in Configure System):

Add sonar-cli, tick Install automatically.

Server Groups Center (SSH2 Easy):

Group Name: docker-group, SSH port 22, user ec2-user, choose password or paste private key.

Add docker-server IP to the group.

Save.

GitHub Repository Preparations
1 – Deploy Key & Webhook
Settings → Deploy keys → Add key → paste the Jenkins public key (github_ssh).

Settings → Webhooks → Add webhook:

Payload URL: http://<JENKINS_IP>:8080/github-webhook/

Content type: application/json

Secret: (optional, if you enable CSRF tokens accordingly)

Events: Just the push event

Save.

GitHub will now POST on each commit.

2 – sonar-project.properties in Repo Root
text
sonar.projectKey=debunker
sonar.projectName=DeBunker
sonar.sources=src
sonar.java.binaries=target/classes
Commit and push.

Creating a Manual Freestyle Job in Jenkins
New Item → Freestyle project → “autobuilder” → OK.

Source Code Management
Select Git.

Repository URL: git@github.com:snowjug/DeBunker.git

Credentials: github_ssh.

Branch: */main.

Build Triggers
Check GitHub hook trigger for GITScm polling.
Jenkins will now build automatically on webhook calls—no polling needed.

Build Environment
Check Prepare SonarQube Scanner environment.
Choose sonar-server from the dropdown.

Build Steps
Execute SonarQube Scanner (appears after plugin install):

Analysis properties (UI textarea):

text
sonar.projectKey=debunker
sonar.projectName=DeBunker
sonar.sources=src
Leave Project Base Dir blank (defaults to workspace).

SSH2 Easy → Remote Script (Add build step):

Target Server: docker-group → docker-server.

Script:

bash
# pull latest code & build image
cd /home/ec2-user/DeBunker || git clone https://github.com/snowjug/DeBunker.git
cd DeBunker && git pull
docker build -t debunker:latest .
docker rm -f debunker || true
docker run -d --name debunker -p 80:80 debunker:latest
Post-Build Actions
(optional) Sonar Quality Gate plugin result, email notifications, etc.

Save the job.

Manual Trigger Walk-Through
Click Build Now → observe Console Output:

Clone → Sonar scan → SSH2 Easy deployment.

Visit SonarQube dashboard—verify debunker analysis appears and Quality Gate status.

Test application: http://<DOCKER_IP>.

GitHub Webhook Live Test
Edit README.md in GitHub → commit to main.

GitHub → Webhooks → Recent Deliveries should show 200 to /github-webhook/.

Jenkins job triggers automatically; console shows new commit hash.

Docker container redeploys with updated image.

Key Operational Tips
Category	Must-Remember Detail	Source
SonarQube token	Only visible once—store in Jenkins credentials immediately	18
Jenkins project properties	Use Analysis properties box in UI or repo file; must define sonar.projectKey, sonar.sources or scan fails	10
Webhook URL format	http(s)://<jenkins>/github-webhook/ (trailing slash required)	20
SSH2 Easy groups	Add one server group, then list multiple hosts to avoid re-entering credentials every time	41
Plugin order	Install plugins first, then refresh Configure System to see new sections	8
CSRF tokens	If Jenkins uses “crumbs,” supply the secret token in GitHub webhook advanced section or disable CSRF for /github-webhook/ path	36
Conclusion
You now have a fully manual, UI-driven CI/CD flow:

Jenkins polls nothing; GitHub webhooks fire builds instantly.

SonarQube evaluates every push through the configured server credentials.

Docker images build and deploy on a dedicated host through SSH2 Easy secure groups.

Follow each section exactly, and your README will guide any teammate—without ever mentioning a declarative Jenkinsfile—from blank AWS account to repeatable three-tier pipeline in under an hour.

Continuous Integration Workflow: GitHub → Jenkins → SonarQube → Docker
Below is a step-by-step guide to achieve a fully integrated CI pipeline such that any code change you make your EC2 development instance is:

Committed & pushed to GitHub

Automatically fetched by Jenkins (autobuilder job) via GitHub webhooks

Analyzed by SonarQube for code quality

Built into a Docker image and deployed to your Docker EC2

1. Prerequisites & Assumptions
You have three EC2 t2.medium instances:

dev-instance where you write code

jenkins-server (Jenkins + GitHub webhook listener)

sonarqube-server (SonarQube on port 9000)

docker-server (Docker daemon on port 80/8081)

Each instance uses Amazon Linux 2, has Java 17 (Corretto) where needed, Docker installed on the Docker host, and Jenkins installed on jenkins-server.

You have a GitHub repo (ssh or https) named DeBunker.

You have SSH keys set up among instances and GitHub deploy key configured.

Security groups allow:

Jenkins: inbound 8080, SSH 22

SonarQube: inbound 9000 from Jenkins

Docker: inbound 80/8081 and SSH 22 from Jenkins/private network

2. Developer Workflow on dev-instance
Edit code under your project folder:

bash
cd ~/DeBunker
# make changes to files (index.html, script.js, Dockerfile, etc.)
Commit & push to GitHub:

bash
git add .
git commit -m "Your feature/bugfix description"
git push origin main
On push, GitHub triggers the webhook to Jenkins.

3. GitHub Webhook Configuration
In your GitHub repo → Settings → Webhooks → Add webhook

Payload URL: http://<JENKINS_IP>:8080/github-webhook/

Content type: application/json

Secret: (optional—if Jenkins is CSRF-protected, configure accordingly)

Events: select Just the push event

Save.

4. Jenkins autobuilder Job Setup
Create new Freestyle job named autobuilder

Source Code Management:

Select Git

Repository URL: git@github.com:snowjug/DeBunker.git

Credentials: your GitHub SSH key credential

Branch Specifier: */main

Build Triggers:

Check GitHub hook trigger for GITScm polling

Build Environment:

Check Prepare SonarQube Scanner → choose your SonarQube server entry

Build Steps:

Execute SonarQube Scanner:

text
sonar.projectKey=debunker
sonar.projectName=DeBunker
sonar.sources=.
Execute Shell (or SSH2 Easy Remote Script to docker-server):

bash
#!/bin/bash
# Ensure latest code on docker host
ssh ec2-user@<DOCKER_IP> '
  cd ~/DeBunker || git clone https://github.com/snowjug/DeBunker.git ~/DeBunker
  cd ~/DeBunker && git pull origin main
  # Build Docker image
  docker build -t debunker:latest .
  # Redeploy
  docker rm -f debunker || true
  docker run -d --name debunker -p 8081:80 debunker:latest
'
Post-Build Actions (optional):

Email or Slack notification on failure/passage.

5. SonarQube Server Configuration
In Jenkins → Manage Jenkins → Configure System → SonarQube Servers

Name: sonar-server

Server URL: http://<SONARQUBE_IP>:9000

Credentials: your SonarQube token (Secret Text)

In GitHub repo root, ensure sonar-project.properties exists:

text
sonar.projectKey=debunker
sonar.projectName=DeBunker
sonar.sources=.
In SonarQube UI → My Account → Security → Generate Token → save into Jenkins credentials.

6. Verification
On code change: edit on dev-instance → git push

Jenkins receives webhook, starts autobuilder →

Clones code

Runs SonarQube analysis → quality gate results

SSHes to Docker host → builds & deploys container

Check:

Jenkins console logs show ➞ “ANALYSIS SUCCESSFUL” and Docker run success

SonarQube dashboard: http://<SONARQUBE_IP>:9000/dashboard?id=debunker

Application URL: http://<DOCKER_IP>:8081

7. Important Points to Remember
Webhook & Credentials: GitHub webhook must point correctly and Jenkins must have SSH and SonarQube tokens configured.

Directory Paths: All Docker commands must run in the directory containing your Dockerfile.

Error Handling: Always use docker rm -f container || true before docker run to avoid “no such container” errors.

Security: Restrict security group rules to only required ports and source IPs.

Quality Gates: Configure SonarQube quality profiles and gates to enforce code standards automatically.
