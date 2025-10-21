1) Launch EC2 and open the right ports

AMI: either Ubuntu 22.04 LTS or Amazon Linux 2023
Instance type: t3.small (or t3.micro for a quick demo)
Key pair: create/download one
Security Group (very important):

22/tcp (SSH) — from your IP

8080/tcp (Jenkins UI) — from your IP (0.0.0.0/0 if it’s just a demo)

8090/tcp (your sample app) — from your IP (or 0.0.0.0/0)

(Optional) Allocate and associate an Elastic IP so your webhook URL doesn’t change.

2) Connect via SSH
chmod 600 your-key.pem
ssh -i your-key.pem ec2-user@EC2_PUBLIC_IP          # Amazon Linux 2023
# or
ssh -i your-key.pem ubuntu@EC2_PUBLIC_IP            # Ubuntu 22.04

3) Install Jenkins, Docker, Git (choose your OS)
A) Amazon Linux 2023 (dnf)
# Update
sudo dnf update -y

# Git + Java 17
sudo dnf install -y git java-17-amazon-corretto

# Jenkins repo + install
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo tee /etc/yum.repos.d/jenkins.repo >/dev/null <<'EOF'
[jenkins]
name=Jenkins-stable
baseurl=https://pkg.jenkins.io/redhat-stable
gpgcheck=1
gpgkey=https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
enabled=1
EOF
sudo dnf install -y jenkins

# Enable & start Jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager

# Docker
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker jenkins
sudo usermod -aG docker ec2-user

# Restart Jenkins to pick up docker group
sudo systemctl restart jenkins

# NOTE: log out and back in so your user gets docker group
exit
# reconnect...
ssh -i your-key.pem ec2-user@EC2_PUBLIC_IP

# Verify docker
docker version
docker run --rm hello-world

B) Ubuntu 22.04
# Update & base tools
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release git

# Java + Jenkins
sudo apt-get install -y openjdk-17-jre
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc >/dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list >/dev/null
sudo apt-get update -y
sudo apt-get install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager

# Docker Engine
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list >/dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Let Jenkins + your user use Docker
sudo usermod -aG docker jenkins
sudo usermod -aG docker $USER
sudo systemctl restart jenkins

# Re-login so your user picks up docker group
exit
# reconnect...
ssh -i your-key.pem ubuntu@EC2_PUBLIC_IP

# Verify
docker version
docker run --rm hello-world


Grab Jenkins initial admin password (you’ll need it in your browser):

sudo cat /var/lib/jenkins/secrets/initialAdminPassword

4) Open Jenkins UI and finish setup

Browse to: http://EC2_PUBLIC_IP:8080

Paste the initialAdminPassword.

Click Install suggested plugins.

Create an admin user.

Confirm plugins like “Pipeline”, “Git”, “Credentials”, “Pipeline: Stage View” are present.

5) Create the sample app repo on the EC2 host
# Workdir
mkdir -p ~/ci-cd-sample && cd ~/ci-cd-sample

# App files
cat > index.html <<'EOF'
<!DOCTYPE html>
<html>
<head><title>CI/CD Demo</title></head>
<body>
<h1>Hello from Jenkins CI/CD!</h1>
<p>Version: v1.0</p>
</body>
</html>
EOF

cat > Dockerfile <<'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
HEALTHCHECK --interval=10s --timeout=2s --retries=3 CMD wget -qO- http://localhost/ || exit 1
EOF


Initialize Git and push to your GitHub (replace username):

git init
git add .
git commit -m "Initial commit: static site + Dockerfile"
git branch -M main
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/ci-cd-sample.git
git push -u origin main

6) Create the Jenkins Pipeline job (with EC2-safe ports)

We’ll run Jenkins UI on 8080 and the app on 8090 to avoid conflicts.

6.1 If repo is private, add GitHub credentials

Jenkins → Manage Jenkins → Credentials → System → Global → Add Credentials

Kind: Username/Password or Secret text (PAT)

ID: github-creds

6.2 New Pipeline job

Jenkins → New Item → Name: ci-cd-demo → Pipeline → OK

Pipeline section:

Definition: Pipeline script from SCM

SCM: Git

Repo URL: https://github.com/YOUR_GITHUB_USERNAME/ci-cd-sample.git

Credentials: select github-creds (if private)

Branch: */main

Script Path: Jenkinsfile

Save.

7) Add the Jenkinsfile (uses port 8090 for app)

From your EC2 shell in the repo folder:

cat > Jenkinsfile <<'EOF'
pipeline {
  agent any

  environment {
    IMAGE_NAME     = "demo/web"
    IMAGE_TAG      = "build-${env.BUILD_NUMBER}"
    CONTAINER_NAME = "ci_web"
    HOST_PORT      = "8090"   // App exposed here to avoid Jenkins 8080 clash
    CONTAINER_PORT = "80"
  }

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
  }

  stages {
    stage('Checkout') {
      steps {
        echo "Checking out source..."
        checkout scm
      }
    }

    stage('Build') {
      steps {
        echo "Building Docker image ${IMAGE_NAME}:${IMAGE_TAG}"
        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
      }
      post {
        success {
          sh 'docker images | head -n 10'
        }
      }
    }

    stage('Test') {
      steps {
        echo "Spinning up test container..."
        sh """
          docker rm -f ${CONTAINER_NAME}-test || true
          docker run -d --name ${CONTAINER_NAME}-test -p 0:${CONTAINER_PORT} ${IMAGE_NAME}:${IMAGE_TAG}

          TEST_PORT="\$(docker inspect ${CONTAINER_NAME}-test \
            --format='{{ (index (index .NetworkSettings.Ports \"${CONTAINER_PORT}/tcp\") 0).HostPort }}')"
          echo "Ephemeral test port: \$TEST_PORT"
          sleep 2

          curl -fsSL http://localhost:\$TEST_PORT | grep -q "Hello from Jenkins CI/CD!"
        """
      }
      post {
        always {
          sh 'docker rm -f ${CONTAINER_NAME}-test || true'
        }
      }
    }

    stage('Deploy') {
      steps {
        echo "Deploying to host on port ${HOST_PORT}..."
        sh """
          docker rm -f ${CONTAINER_NAME} || true
          docker run -d --name ${CONTAINER_NAME} -p ${HOST_PORT}:${CONTAINER_PORT} ${IMAGE_NAME}:${IMAGE_TAG}
          sleep 2
          curl -fsSL http://localhost:${HOST_PORT} | grep -q "Hello from Jenkins CI/CD!"
        """
      }
    }
  }

  post {
    success {
      echo "SUCCESS: App available at http://<EC2_PUBLIC_IP>:${HOST_PORT}"
    }
    failure {
      echo "FAILED: see logs below"
      sh 'docker ps -a || true'
    }
    always {
      echo "Build ${env.BUILD_NUMBER} finished."
    }
  }
}
EOF

git add Jenkinsfile
git commit -m "Add Jenkinsfile for EC2 (deploy on port 8090)"
git push

8) Run and verify

Trigger: Jenkins job → Build Now → watch Console Output.
On EC2 host:

docker ps                       # should show 'ci_web' on 0.0.0.0:8090->80/tcp
curl -i http://localhost:8090


From your laptop browser:
http://EC2_PUBLIC_IP:8090 should show the page with Version: v1.0.

9) Make a change to see CI/CD in action
cd ~/ci-cd-sample
sed -i 's/v1.0/v1.1/' index.html
git add index.html
git commit -m "Bump version to v1.1"
git push


Rebuild (or set webhook below). Verify the page now shows v1.1 at http://EC2_PUBLIC_IP:8090.

10) (Optional) Auto-trigger builds via GitHub Webhook

Jenkins job → Configure → Build Triggers → check GitHub hook trigger for GITScm polling → Save.

GitHub repo → Settings → Webhooks → Add webhook

Payload URL: http://EC2_PUBLIC_IP:8080/github-webhook/

Content type: application/json

Event: Just the push event

Add webhook

Ensure your Security Group allows the source (GitHub’s hooks) to reach 8080. For a quick demo, allow 0.0.0.0/0.

11) Commands you might need for troubleshooting on EC2
# Jenkins admin password (if you forgot)
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Service status/logs
sudo systemctl status jenkins --no-pager
sudo journalctl -u jenkins -f

sudo systemctl status docker --no-pager
sudo journalctl -u docker -f

# Docker perms error fix (run, then restart Jenkins, then re-login user if needed)
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# Free the app port if needed
sudo lsof -i :8090 || true
docker rm -f ci_web || true

# Clean up old images/containers
docker ps -a
docker images
docker system prune -f

12) What to put in solution.md

Jenkinsfile (the one above)

Stage purposes (Build compiles image, Test validates via curl, Deploy runs fixed-port container)

Exact commands you ran (OS install, Jenkins/Docker setup, git pushes)

Screenshots: Jenkins Stage View green; docker ps showing ci_web; browser hitting http://EC2_PUBLIC_IP:8090

Issues & fixes (e.g., Docker permissions, SG ports closed, port conflicts)

Interview Q&A (quickfire)

Q: How do declarative pipelines streamline the CI/CD process vs scripted?

Opinionated structure (pipeline { stages { … } }) → consistent, readable, reviewable

Built-ins (post/when/options/environment) reduce custom Groovy glue

Better validation and Stage View/Blue Ocean UX

Guardrails (fewer foot-guns); still allows script {} when needed

Q: Why break into Build/Test/Deploy stages?

Observability: isolate failures, faster RCA

Parallelism: run tests in parallel subsections

Quality gates: enforce tests/scans before deploy

Reusability: artifacts/tags flow stage-to-stage

Fail fast: save compute + developer time

If you want me to generate a ready-to-paste solution.md filled with your EC2 public IP and the exact commands you used (Ubuntu or AL2023), tell me which AMI you picked—I’ll spit out the final file.