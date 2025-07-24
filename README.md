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
