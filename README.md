# Complete CI/CD Documentation
## GitHub Actions & Jenkins with AWS EC2
### Python and Golang Applications - Production Ready Guide

---

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Environment Setup](#environment-setup)
5. [CI/CD using GitHub Actions + AWS EC2](#github-actions-cicd)
6. [CI/CD using Jenkins + AWS EC2](#jenkins-cicd)
7. [Testing & Quality Gates](#testing-quality)
8. [Deployment Strategies](#deployment-strategies)
9. [Service Management with systemd](#systemd-service)
10. [Security & Secrets Management](#security)
11. [Monitoring & Logging](#monitoring)
12. [Rollback & Recovery](#rollback)
13. [Best Practices](#best-practices)
14. [Troubleshooting](#troubleshooting)
15. [Cost Optimization](#cost-optimization)
16. [Comparison: GitHub Actions vs Jenkins](#comparison)

---

## 1. Overview {#overview}

This comprehensive guide provides complete end-to-end CI/CD pipelines for deploying Python and Golang applications on AWS EC2 using both GitHub Actions and Jenkins. It covers production-ready configurations, security best practices, monitoring, and deployment strategies.

**What You'll Learn:**
- Setting up automated CI/CD pipelines from scratch
- Implementing blue-green and rolling deployments
- Configuring automated testing and quality gates
- Managing secrets and security best practices
- Implementing monitoring, logging, and alerting
- Creating rollback strategies for failed deployments

---

## 2. Architecture {#architecture}

### GitHub Actions Architecture
```
Developer → Git Push → GitHub Repository
                              ↓
                        GitHub Actions
                              ↓
                    [Build] → [Test] → [Deploy]
                              ↓
                         AWS EC2 Instance
                              ↓
                    [Application Running]
```

### Jenkins Architecture
```
Developer → Git Push → GitHub Repository (Webhook)
                              ↓
                        Jenkins Server
                              ↓
              [Build] → [Test] → [Quality] → [Deploy]
                              ↓
                         AWS EC2 Instance
                              ↓
                    [Application Running]
```

### Multi-Environment Architecture
```
                    GitHub Repository
                            ↓
                    CI/CD Pipeline
                    /       |       \
                   /        |        \
            Development  Staging  Production
            (EC2-Dev)  (EC2-Stage) (EC2-Prod)
```

---

## 3. Prerequisites {#prerequisites}

### AWS Requirements
- **EC2 Instance**: Ubuntu 20.04 LTS or 22.04 LTS
- **Instance Type**: t2.micro (dev) / t2.small+ (prod)
- **Security Group Rules**:
  - Port 22: SSH (restricted to your IP)
  - Port 80: HTTP
  - Port 443: HTTPS
  - Port 8080: Application (or custom port)
- **PEM Key**: Downloaded and secured
- **IAM Role** (optional): For S3, CloudWatch access
- **Elastic IP** (recommended): For stable DNS

### GitHub Requirements
- GitHub repository with admin access
- GitHub Secrets configured:
  - `EC2_HOST`: EC2 public IP or domain
  - `EC2_USER`: ubuntu (or ec2-user)
  - `EC2_SSH_KEY`: Private key content
  - `APP_PORT`: Application port number

### Jenkins Requirements
- Jenkins server (EC2 or on-premises)
- Jenkins version: 2.400+ recommended
- Required plugins:
  - Git Plugin
  - Pipeline Plugin
  - SSH Agent Plugin
  - Credentials Plugin
  - GitHub Integration Plugin
  - Workspace Cleanup Plugin
- GitHub webhook configured

### Development Tools
- Git 2.30+
- Python 3.10+ or Go 1.22+
- Docker (optional, for containerized builds)
- AWS CLI configured

---

## 4. Environment Setup {#environment-setup}

### EC2 Instance Setup

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y git curl wget build-essential

# Python Setup
sudo apt install -y python3 python3-pip python3-venv
python3 --version

# Go Setup
wget https://go.dev/dl/go1.22.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
go version

# Create application directory
sudo mkdir -p /var/www/app
sudo chown -R ubuntu:ubuntu /var/www/app

# Install Nginx (reverse proxy)
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

### Nginx Configuration (Reverse Proxy)

```bash
# Create Nginx config
sudo nano /etc/nginx/sites-available/app

# Add configuration
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

# Enable site
sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### SSL/TLS Setup with Let's Encrypt

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtain certificate
sudo certbot --nginx -d your-domain.com

# Auto-renewal is configured automatically
sudo certbot renew --dry-run
```

---

## 5. CI/CD using GitHub Actions + AWS EC2 {#github-actions-cicd}

### Python Application - Complete Workflow

```yaml
name: Python CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  PYTHON_VERSION: '3.10'
  APP_NAME: python-app

jobs:
  # Job 1: Code Quality & Linting
  quality:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install -r requirements.txt
          pip install flake8 black pylint bandit safety

      - name: Code formatting check (Black)
        run: black --check .

      - name: Linting (Flake8)
        run: flake8 . --max-line-length=100 --exclude=venv,__pycache__

      - name: Static analysis (Pylint)
        run: pylint **/*.py || true

      - name: Security scan (Bandit)
        run: bandit -r . -f json -o bandit-report.json || true

      - name: Dependency vulnerability check
        run: safety check

  # Job 2: Testing
  test:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov pytest-xdist

      - name: Run unit tests
        run: pytest tests/unit -v --cov=src --cov-report=xml --cov-report=html

      - name: Run integration tests
        run: pytest tests/integration -v

      - name: Upload coverage reports
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml
          flags: unittests

  # Job 3: Build
  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Create deployment package
        run: |
          mkdir -p dist
          cp -r src dist/
          cp requirements.txt dist/
          cp -r static dist/ || true
          cp -r templates dist/ || true

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: python-app
          path: dist/

  # Job 4: Deploy to EC2
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v3
        with:
          name: python-app

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          port: 22
          script: |
            # Navigate to app directory
            cd /var/www/app
            
            # Backup current version
            if [ -d "current" ]; then
              timestamp=$(date +%Y%m%d_%H%M%S)
              mv current backup_$timestamp
            fi
            
            # Pull latest code
            git pull origin main
            
            # Create virtual environment
            python3 -m venv venv
            source venv/bin/activate
            
            # Install dependencies
            pip install --upgrade pip
            pip install -r requirements.txt
            
            # Run database migrations (if applicable)
            # python manage.py migrate
            
            # Restart application
            sudo systemctl restart python-app
            sudo systemctl status python-app
            
            # Health check
            sleep 5
            curl -f http://localhost:8080/health || exit 1

      - name: Deployment notification
        if: always()
        run: |
          if [ ${{ job.status }} == 'success' ]; then
            echo "✅ Deployment successful!"
          else
            echo "❌ Deployment failed!"
          fi
```

### Golang Application - Complete Workflow

```yaml
name: Go CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  GO_VERSION: '1.22'
  APP_NAME: go-app

jobs:
  # Job 1: Code Quality
  quality:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Install dependencies
        run: go mod download

      - name: Format check
        run: |
          if [ "$(gofmt -s -l . | wc -l)" -gt 0 ]; then
            echo "Code is not formatted. Run 'gofmt -s -w .'"
            exit 1
          fi

      - name: Vet
        run: go vet ./...

      - name: Static analysis (staticcheck)
        run: |
          go install honnef.co/go/tools/cmd/staticcheck@latest
          staticcheck ./...

      - name: Security scan (gosec)
        run: |
          go install github.com/securego/gosec/v2/cmd/gosec@latest
          gosec -fmt json -out gosec-report.json ./...

  # Job 2: Testing
  test:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Run tests
        run: go test -v -race -coverprofile=coverage.out -covermode=atomic ./...

      - name: Coverage report
        run: go tool cover -html=coverage.out -o coverage.html

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.out
          flags: unittests

  # Job 3: Build
  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: Build binary
        run: |
          CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
            -ldflags="-w -s -X main.Version=$(git describe --tags --always) -X main.BuildTime=$(date -u +%Y%m%d.%H%M%S)" \
            -o ${{ env.APP_NAME }} \
            ./cmd/main.go

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: go-app-binary
          path: ${{ env.APP_NAME }}

  # Job 4: Deploy
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v3
        with:
          name: go-app-binary

      - name: Copy binary to EC2
        uses: appleboy/scp-action@v0.1.4
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          source: ${{ env.APP_NAME }}
          target: /home/ubuntu/app/
          overwrite: true

      - name: Deploy and restart
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd /home/ubuntu/app
            
            # Backup current binary
            if [ -f go-app ]; then
              timestamp=$(date +%Y%m%d_%H%M%S)
              mv go-app go-app.backup.$timestamp
            fi
            
            # Make binary executable
            chmod +x ${{ env.APP_NAME }}
            
            # Restart service
            sudo systemctl restart go-app
            sudo systemctl status go-app
            
            # Health check
            sleep 3
            curl -f http://localhost:8080/health || exit 1
            
            # Cleanup old backups (keep last 5)
            ls -t go-app.backup.* | tail -n +6 | xargs rm -f
```

---

## 6. CI/CD using Jenkins + AWS EC2 {#jenkins-cicd}

### Jenkins Server Setup

```bash
# Install Java
sudo apt update
sudo apt install -y openjdk-11-jdk

# Add Jenkins repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt update
sudo apt install -y jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Python Application - Jenkinsfile

```groovy
pipeline {
    agent any
    
    environment {
        APP_NAME = 'python-app'
        APP_DIR = '/var/www/app'
        VENV_DIR = "${APP_DIR}/venv"
        PYTHON_VERSION = '3.10'
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }
        
        stage('Setup Environment') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install flake8 pytest pytest-cov black
                '''
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('Vet') {
                    steps {
                        sh 'go vet ./...'
                    }
                }
                stage('Security') {
                    steps {
                        sh '''
                            go install github.com/securego/gosec/v2/cmd/gosec@latest
                            gosec -fmt json -out gosec-report.json ./... || true
                        '''
                    }
                }
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    go test -v -race -coverprofile=coverage.out -covermode=atomic ./...
                    go tool cover -html=coverage.out -o coverage.html
                '''
            }
            post {
                always {
                    publishHTML([
                        reportDir: '.',
                        reportFiles: 'coverage.html',
                        reportName: 'Go Coverage Report'
                    ])
                }
            }
        }
        
        stage('Build') {
            steps {
                sh '''
                    VERSION=$(git describe --tags --always)
                    BUILD_TIME=$(date -u +%Y%m%d.%H%M%S)
                    
                    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
                        -ldflags="-w -s -X main.Version=$VERSION -X main.BuildTime=$BUILD_TIME" \
                        -o ${APP_NAME} \
                        ./cmd/main.go
                    
                    # Create checksum
                    sha256sum ${APP_NAME} > ${APP_NAME}.sha256
                '''
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        # Copy binary and checksum
                        scp -o StrictHostKeyChecking=no ${APP_NAME} ubuntu@${EC2_HOST}:${APP_DIR}/
                        scp ${APP_NAME}.sha256 ubuntu@${EC2_HOST}:${APP_DIR}/
                        
                        # Deploy
                        ssh ubuntu@${EC2_HOST} "
                            set -e
                            cd ${APP_DIR}
                            
                            # Verify checksum
                            sha256sum -c ${APP_NAME}.sha256
                            
                            # Backup current binary
                            if [ -f go-app ]; then
                                timestamp=\$(date +%Y%m%d_%H%M%S)
                                mv go-app go-app.backup.\$timestamp
                            fi
                            
                            # Make executable
                            chmod +x ${APP_NAME}
                            mv ${APP_NAME} go-app
                            
                            # Restart service
                            sudo systemctl restart go-app
                            
                            # Health check
                            sleep 3
                            curl -f http://localhost:8080/health || exit 1
                            
                            # Cleanup old backups
                            ls -t go-app.backup.* | tail -n +6 | xargs rm -f
                        "
                    '''
                }
            }
        }
        
        stage('Smoke Tests') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    curl -f http://${EC2_HOST}/health
                    curl -f http://${EC2_HOST}/api/version
                '''
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed! Rolling back...'
            sshagent(credentials: ['ec2-ssh-key']) {
                sh '''
                    ssh ubuntu@${EC2_HOST} "
                        cd ${APP_DIR}
                        if [ -f go-app.backup.* ]; then
                            latest_backup=\$(ls -t go-app.backup.* | head -1)
                            cp \$latest_backup go-app
                            sudo systemctl restart go-app
                        fi
                    "
                '''
            }
        }
        always {
            cleanWs()
        }
    }
}
```

---

## 7. Testing & Quality Gates {#testing-quality}

### Testing Strategy

#### Test Pyramid
```
           /\
          /  \        E2E Tests (10%)
         /____\
        /      \      Integration Tests (20%)
       /________\
      /          \    Unit Tests (70%)
     /____________\
```

### Python Testing Configuration

**pytest.ini**
```ini
[pytest]
minversion = 6.0
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = 
    -v
    --strict-markers
    --cov=src
    --cov-report=html
    --cov-report=xml
    --cov-report=term-missing
    --cov-fail-under=80
markers =
    unit: Unit tests
    integration: Integration tests
    e2e: End-to-end tests
    slow: Slow running tests
```

**requirements-dev.txt**
```
pytest>=7.4.0
pytest-cov>=4.1.0
pytest-xdist>=3.3.0
pytest-mock>=3.11.0
black>=23.7.0
flake8>=6.1.0
pylint>=2.17.0
bandit>=1.7.5
safety>=2.3.0
mypy>=1.5.0
```

### Go Testing Configuration

**Makefile**
```makefile
.PHONY: test test-unit test-integration test-coverage lint fmt security

test: test-unit test-integration

test-unit:
	go test -v -race -short ./...

test-integration:
	go test -v -race -run Integration ./...

test-coverage:
	go test -v -race -coverprofile=coverage.out -covermode=atomic ./...
	go tool cover -html=coverage.out -o coverage.html
	go tool cover -func=coverage.out

lint:
	golangci-lint run ./...

fmt:
	gofmt -s -w .
	go mod tidy

security:
	gosec -fmt json -out gosec-report.json ./...
	
bench:
	go test -bench=. -benchmem ./...
```

### Quality Gates Configuration

**GitHub Actions - Quality Gate**
```yaml
name: Quality Gate

on: [push, pull_request]

jobs:
  quality-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Code coverage check
        run: |
          coverage=$(python -m pytest --cov=src --cov-report=json | jq '.totals.percent_covered')
          if (( $(echo "$coverage < 80" | bc -l) )); then
            echo "Coverage $coverage% is below 80%"
            exit 1
          fi
      
      - name: Complexity check
        run: |
          pip install radon
          radon cc src -a -nb
      
      - name: Technical debt check
        run: |
          pip install pylint
          score=$(pylint src --exit-zero | grep "rated at" | awk '{print $7}' | cut -d'/' -f1)
          if (( $(echo "$score < 8.0" | bc -l) )); then
            echo "Code quality score $score is below 8.0"
            exit 1
          fi
```

---

## 8. Deployment Strategies {#deployment-strategies}

### Blue-Green Deployment

**Architecture**
```
Load Balancer
     |
     |---- Blue Environment (Current)
     |---- Green Environment (New)
```

**Implementation - Python**
```bash
#!/bin/bash
# blue-green-deploy.sh

APP_DIR="/var/www/app"
BLUE_PORT=8080
GREEN_PORT=8081
HEALTH_CHECK_URL="http://localhost"

# Deploy to green environment
deploy_green() {
    echo "Deploying to GREEN environment..."
    cd $APP_DIR/green
    git pull origin main
    source venv/bin/activate
    pip install -r requirements.txt
    
    # Start green instance
    PORT=$GREEN_PORT python app.py &
    GREEN_PID=$!
    
    # Wait for startup
    sleep 5
    
    # Health check
    if curl -f $HEALTH_CHECK_URL:$GREEN_PORT/health; then
        echo "GREEN environment healthy"
        return 0
    else
        echo "GREEN environment unhealthy"
        kill $GREEN_PID
        return 1
    fi
}

# Switch traffic
switch_traffic() {
    echo "Switching traffic to GREEN..."
    
    # Update Nginx to point to green
    sudo sed -i "s/:$BLUE_PORT/:$GREEN_PORT/" /etc/nginx/sites-available/app
    sudo nginx -t && sudo nginx -s reload
    
    # Stop blue environment
    sleep 10
    pkill -f "PORT=$BLUE_PORT"
    
    echo "Traffic switched successfully"
}

# Rollback
rollback() {
    echo "Rolling back to BLUE..."
    sudo sed -i "s/:$GREEN_PORT/:$BLUE_PORT/" /etc/nginx/sites-available/app
    sudo nginx -s reload
}

# Main execution
if deploy_green; then
    switch_traffic
else
    rollback
    exit 1
fi
```

### Rolling Deployment

**GitHub Actions - Rolling Deployment**
```yaml
name: Rolling Deployment

on:
  push:
    branches: [main]

jobs:
  rolling-deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        server: 
          - ec2-1.example.com
          - ec2-2.example.com
          - ec2-3.example.com
      max-parallel: 1  # Deploy one at a time
      
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to ${{ matrix.server }}
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ matrix.server }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            # Remove from load balancer
            aws elbv2 deregister-targets \
              --target-group-arn ${{ secrets.TG_ARN }} \
              --targets Id=$(ec2-metadata --instance-id | cut -d ' ' -f 2)
            
            # Wait for connection draining
            sleep 30
            
            # Deploy
            cd /var/www/app
            git pull origin main
            sudo systemctl restart app
            
            # Health check
            sleep 10
            curl -f http://localhost:8080/health || exit 1
            
            # Re-register to load balancer
            aws elbv2 register-targets \
              --target-group-arn ${{ secrets.TG_ARN }} \
              --targets Id=$(ec2-metadata --instance-id | cut -d ' ' -f 2)
            
            # Wait for healthy status
            sleep 20
```

### Canary Deployment

**Canary Strategy with Nginx**
```nginx
# Nginx canary configuration
upstream backend {
    server 10.0.1.10:8080 weight=9;  # 90% traffic - stable
    server 10.0.1.11:8080 weight=1;  # 10% traffic - canary
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
    }
}
```

**Automated Canary Script**
```bash
#!/bin/bash
# canary-deploy.sh

CANARY_HOST="10.0.1.11"
STABLE_HOST="10.0.1.10"
CANARY_WEIGHT=1
STABLE_WEIGHT=9
ERROR_THRESHOLD=5

# Deploy canary
deploy_canary() {
    ssh ubuntu@$CANARY_HOST "
        cd /var/www/app
        git pull origin main
        sudo systemctl restart app
    "
}

# Monitor canary
monitor_canary() {
    for i in {1..10}; do
        error_rate=$(curl -s http://$CANARY_HOST/metrics | jq '.error_rate')
        
        if (( $(echo "$error_rate > $ERROR_THRESHOLD" | bc -l) )); then
            echo "Canary error rate too high: $error_rate%"
            rollback_canary
            exit 1
        fi
        
        echo "Canary check $i/10: Error rate $error_rate%"
        sleep 60
    done
}

# Promote canary
promote_canary() {
    echo "Promoting canary to production..."
    ssh ubuntu@$STABLE_HOST "
        cd /var/www/app
        git pull origin main
        sudo systemctl restart app
    "
}

# Rollback canary
rollback_canary() {
    echo "Rolling back canary..."
    ssh ubuntu@$CANARY_HOST "
        cd /var/www/app
        git checkout HEAD~1
        sudo systemctl restart app
    "
}

# Execute
deploy_canary
if monitor_canary; then
    promote_canary
    echo "✅ Canary deployment successful"
else
    echo "❌ Canary deployment failed"
    exit 1
fi
```

---

## 9. Service Management with systemd {#systemd-service}

### Python Application Service

**/etc/systemd/system/python-app.service**
```ini
[Unit]
Description=Python Web Application
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/var/www/app
Environment="PATH=/var/www/app/venv/bin"
Environment="PYTHONUNBUFFERED=1"
EnvironmentFile=/var/www/app/.env

# Main process
ExecStart=/var/www/app/venv/bin/python app.py

# Restart policy
Restart=always
RestartSec=10
StartLimitInterval=200
StartLimitBurst=5

# Resource limits
MemoryLimit=512M
CPUQuota=50%

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/www/app/logs /var/www/app/data

# Logging
StandardOutput=append:/var/log/python-app/output.log
StandardError=append:/var/log/python-app/error.log

[Install]
WantedBy=multi-user.target
```

### Golang Application Service

**/etc/systemd/system/go-app.service**
```ini
[Unit]
Description=Go Application Service
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/app
EnvironmentFile=/home/ubuntu/app/.env

# Main process
ExecStart=/home/ubuntu/app/go-app

# Pre-start checks
ExecStartPre=/usr/bin/test -f /home/ubuntu/app/go-app
ExecStartPre=/usr/bin/chmod +x /home/ubuntu/app/go-app

# Graceful shutdown
KillMode=mixed
KillSignal=SIGTERM
TimeoutStopSec=30

# Restart policy
Restart=on-failure
RestartSec=5
StartLimitInterval=100
StartLimitBurst=3

# Resource limits
MemoryLimit=1G
CPUQuota=75%
TasksMax=100

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=true
ReadOnlyPaths=/
ReadWritePaths=/home/ubuntu/app/data /home/ubuntu/app/logs

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=go-app

[Install]
WantedBy=multi-user.target
```

### Service Management Commands

```bash
# Enable service
sudo systemctl enable python-app

# Start service
sudo systemctl start python-app

# Check status
sudo systemctl status python-app

# View logs
sudo journalctl -u python-app -f

# Restart service
sudo systemctl restart python-app

# Stop service
sudo systemctl stop python-app

# Reload systemd
sudo systemctl daemon-reload

# View service configuration
systemctl cat python-app
```

### Multi-Instance Services

**/etc/systemd/system/go-app@.service**
```ini
[Unit]
Description=Go Application Instance %i
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/app
Environment="PORT=%i"
ExecStart=/home/ubuntu/app/go-app
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
# Start multiple instances
sudo systemctl start go-app@8080
sudo systemctl start go-app@8081
sudo systemctl start go-app@8082

# Enable all instances
sudo systemctl enable go-app@{8080,8081,8082}
```

---

## 10. Security & Secrets Management {#security}

### GitHub Secrets Setup

**Required Secrets:**
```
EC2_HOST          # EC2 public IP or domain
EC2_USER          # ubuntu or ec2-user
EC2_SSH_KEY       # Private SSH key content
DB_PASSWORD       # Database password
API_KEY           # Third-party API keys
JWT_SECRET        # JWT signing secret
AWS_ACCESS_KEY    # AWS credentials (if needed)
AWS_SECRET_KEY    # AWS credentials (if needed)
```

**Adding Secrets via GitHub CLI:**
```bash
# Install GitHub CLI
brew install gh  # macOS
# or
sudo apt install gh  # Ubuntu

# Login
gh auth login

# Add secrets
gh secret set EC2_HOST
gh secret set EC2_USER
gh secret set EC2_SSH_KEY < ~/.ssh/id_rsa

# List secrets
gh secret list
```

### Environment Variables Management

**.env Template**
```bash
# Application
APP_NAME=myapp
APP_ENV=production
APP_PORT=8080
DEBUG=false

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=appdb
DB_USER=appuser
DB_PASSWORD=${DB_PASSWORD}

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# Security
JWT_SECRET=${JWT_SECRET}
API_KEY=${API_KEY}

# AWS
AWS_REGION=us-east-1
S3_BUCKET=my-app-bucket

# Monitoring
SENTRY_DSN=${SENTRY_DSN}
```

### Secure Secret Injection

**GitHub Actions - Secret Injection**
```yaml
- name: Deploy with secrets
  uses: appleboy/ssh-action@v1.0.0
  with:
    host: ${{ secrets.EC2_HOST }}
    username: ${{ secrets.EC2_USER }}
    key: ${{ secrets.EC2_SSH_KEY }}
    envs: DB_PASSWORD,JWT_SECRET,API_KEY
    script: |
      cd /var/www/app
      
      # Create .env file
      cat > .env << EOF
      DB_PASSWORD=$DB_PASSWORD
      JWT_SECRET=$JWT_SECRET
      API_KEY=$API_KEY
      EOF
      
      # Secure permissions
      chmod 600 .env
      
      sudo systemctl restart app
  env:
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
    JWT_SECRET: ${{ secrets.JWT_SECRET }}
    API_KEY: ${{ secrets.API_KEY }}
```

### AWS Secrets Manager Integration

**Python - Fetch Secrets**
```python
import boto3
import json
from botocore.exceptions import ClientError

def get_secret(secret_name, region_name="us-east-1"):
    """Fetch secret from AWS Secrets Manager"""
    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name=region_name
    )
    
    try:
        response = client.get_secret_value(SecretId=secret_name)
        return json.loads(response['SecretString'])
    except ClientError as e:
        raise e

# Usage
secrets = get_secret('production/myapp/secrets')
db_password = secrets['DB_PASSWORD']
```

**Go - Fetch Secrets**
```go
package main

import (
    "encoding/json"
    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/secretsmanager"
)

func getSecret(secretName string) (map[string]string, error) {
    sess := session.Must(session.NewSession())
    svc := secretsmanager.New(sess)
    
    input := &secretsmanager.GetSecretValueInput{
        SecretId: aws.String(secretName),
    }
    
    result, err := svc.GetSecretValue(input)
    if err != nil {
        return nil, err
    }
    
    var secrets map[string]string
    json.Unmarshal([]byte(*result.SecretString), &secrets)
    return secrets, nil
}
```

### SSH Key Management

**Generate SSH Key**
```bash
# Generate new key pair
ssh-keygen -t ed25519 -C "cicd@github" -f ~/.ssh/github_cicd

# Add public key to EC2
cat ~/.ssh/github_cicd.pub | ssh ubuntu@EC2_IP "cat >> ~/.ssh/authorized_keys"

# Add private key to GitHub Secrets
cat ~/.ssh/github_cicd  # Copy this to GitHub Secrets
```

### Security Best Practices

**EC2 Security Group Rules**
```bash
# SSH - Restricted to your IP
Protocol: TCP
Port: 22
Source: YOUR_IP/32

# HTTP
Protocol: TCP
Port: 80
Source: 0.0.0.0/0

# HTTPS
Protocol: TCP
Port: 443
Source: 0.0.0.0/0

# Application (internal only)
Protocol: TCP
Port: 8080
Source: Security Group ID (self-reference)
```

**Fail2ban Configuration**
```bash
# Install fail2ban
sudo apt install fail2ban

# Configure SSH protection
sudo nano /etc/fail2ban/jail.local

[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600

# Restart fail2ban
sudo systemctl restart fail2ban
```

---

## 11. Monitoring & Logging {#monitoring}

### Application Logging

**Python - Structured Logging**
```python
import logging
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno
        }
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        return json.dumps(log_data)

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    handlers=[
        logging.FileHandler('/var/log/app/app.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

# Usage
logger.info("Application started", extra={'port': 8080})
logger.error("Database connection failed", exc_info=True)
```

**Go - Structured Logging with Zap**
```go
package main

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

func initLogger() *zap.Logger {
    config := zap.NewProductionConfig()
    config.OutputPaths = []string{
        "stdout",
        "/var/log/app/app.log",
    }
    config.EncoderConfig.TimeKey = "timestamp"
    config.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
    
    logger, _ := config.Build()
    return logger
}

func main() {
    logger := initLogger()
    defer logger.Sync()
    
    logger.Info("Application started",
        zap.String("version", "1.0.0"),
        zap.Int("port", 8080),
    )
    
    logger.Error("Database connection failed",
        zap.Error(err),
        zap.String("host", "localhost"),
    )
}
```

### Health Check Endpoints

**Python - Flask Health Check**
```python
from flask import Flask, jsonify
import psutil
import time

app = Flask(__name__)
start_time = time.time()

@app.route('/health')
def health():
    return jsonify({
        'status': 'healthy',
        'timestamp': time.time()
    }), 200

@app.route('/ready')
def ready():
    # Check database connection
    try:
        db.session.execute('SELECT 1')
        db_status = 'connected'
    except:
        db_status = 'disconnected'
        return jsonify({'status': 'not ready'}), 503
    
    return jsonify({
        'status': 'ready',
        'database': db_status
    }), 200

@app.route('/metrics')
def metrics():
    uptime = time.time() - start_time
    
    return jsonify({
        'uptime_seconds': uptime,
        'cpu_percent': psutil.cpu_percent(),
        'memory_percent': psutil.virtual_memory().percent,
        'disk_percent': psutil.disk_usage('/').percent
    }), 200
```

**Go - Health Check Handler**
```go
package main

import (
    "encoding/json"
    "net/http"
    "time"
)

var startTime = time.Now()

type HealthResponse struct {
    Status    string  `json:"status"`
    Timestamp int64   `json:"timestamp"`
}

type MetricsResponse struct {
    UptimeSeconds float64 `json:"uptime_seconds"`
    Goroutines    int     `json:"goroutines"`
    MemoryMB      uint64  `json:"memory_mb"`
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    response := HealthResponse{
        Status:    "healthy",
        Timestamp: time.Now().Unix(),
    }
    json.NewEncoder(w).Encode(response)
}

func metricsHandler(w http.ResponseWriter, r *http.Request) {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    
    response := MetricsResponse{
        UptimeSeconds: time.Since(startTime).Seconds(),
        Goroutines:    runtime.NumGoroutine(),
        MemoryMB:      m.Alloc / 1024 / 1024,
    }
    json.NewEncoder(w).Encode(response)
}

func main() {
    http.HandleFunc("/health", healthHandler)
    http.HandleFunc("/metrics", metricsHandler)
    http.ListenAndServe(":8080", nil)
}
```

### Prometheus Integration

**prometheus.yml**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'python-app'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/metrics'
    
  - job_name: 'go-app'
    static_configs:
      - targets: ['localhost:8081']
    
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
```

**Install Prometheus & Grafana**
```bash
# Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
tar xvfz prometheus-*.tar.gz
cd prometheus-*
./prometheus --config.file=prometheus.yml

# Node Exporter (system metrics)
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.0/node_exporter-1.6.0.linux-amd64.tar.gz
tar xvfz node_exporter-*.tar.gz
cd node_exporter-*
./node_exporter

# Grafana
sudo apt-get install -y software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo apt-get update
sudo apt-get install grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# Access Grafana at http://localhost:3000 (admin/admin)
```

### CloudWatch Integration

**Install CloudWatch Agent**
```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb

# Configure CloudWatch agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -s \
    -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json
```

**CloudWatch Configuration**
```json
{
  "metrics": {
    "namespace": "MyApp",
    "metrics_collected": {
      "cpu": {
        "measurement": [
          {"name": "cpu_usage_idle", "rename": "CPU_IDLE", "unit": "Percent"},
          "cpu_usage_iowait"
        ],
        "totalcpu": false
      },
      "disk": {
        "measurement": [
          {"name": "used_percent", "rename": "DISK_USED", "unit": "Percent"}
        ],
        "resources": ["/"]
      },
      "mem": {
        "measurement": [
          {"name": "mem_used_percent", "rename": "MEM_USED", "unit": "Percent"}
        ]
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/app/*.log",
            "log_group_name": "/aws/ec2/myapp",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```

### Log Aggregation

**ELK Stack (Elasticsearch, Logstash, Kibana)**
```bash
# Install Elasticsearch
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
sudo apt update
sudo apt install elasticsearch

# Start Elasticsearch
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch

# Install Logstash
sudo apt install logstash

# Logstash configuration
sudo nano /etc/logstash/conf.d/app.conf

input {
  file {
    path => "/var/log/app/*.log"
    type => "app-log"
    codec => json
  }
}

filter {
  if [type] == "app-log" {
    json {
      source => "message"
    }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "app-logs-%{+YYYY.MM.dd}"
  }
}

# Start Logstash
sudo systemctl start logstash

# Install Kibana
sudo apt install kibana
sudo systemctl start kibana
sudo systemctl enable kibana

# Access Kibana at http://localhost:5601
```

### Alerting Configuration

**Prometheus Alertmanager**
```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: 'your-password'

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'team-emails'

receivers:
  - name: 'team-emails'
    email_configs:
      - to: 'team@example.com'
        headers:
          Subject: '🚨 {{ .GroupLabels.alertname }}'
  
  - name: 'slack-notifications'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

**Alert Rules**
```yaml
# alert.rules.yml
groups:
  - name: application_alerts
    interval: 30s
    rules:
      - alert: HighCPUUsage
        expr: cpu_usage > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage is above 80% for 5 minutes"
      
      - alert: HighMemoryUsage
        expr: memory_usage > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
      
      - alert: ApplicationDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Application is down"
          description: "Application has been down for 1 minute"
      
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is above 5%"
```

---

## 12. Rollback & Recovery {#rollback}

### Automated Rollback Strategy

**Rollback Script - Python**
```bash
#!/bin/bash
# rollback.sh

APP_DIR="/var/www/app"
BACKUP_DIR="$APP_DIR/backups"
LOG_FILE="/var/log/rollback.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

list_backups() {
    echo "Available backups:"
    ls -lht $BACKUP_DIR | grep backup_
}

rollback_to_version() {
    local version=$1
    
    if [ -z "$version" ]; then
        log "ERROR: No version specified"
        list_backups
        exit 1
    fi
    
    if [ ! -d "$BACKUP_DIR/backup_$version" ]; then
        log "ERROR: Backup version $version not found"
        list_backups
        exit 1
    fi
    
    log "Starting rollback to version $version"
    
    # Stop application
    log "Stopping application..."
    sudo systemctl stop python-app
    
    # Backup current version
    timestamp=$(date +%Y%m%d_%H%M%S)
    cp -r $APP_DIR/current $BACKUP_DIR/pre_rollback_$timestamp
    
    # Restore backup
    log "Restoring backup..."
    rm -rf $APP_DIR/current
    cp -r $BACKUP_DIR/backup_$version $APP_DIR/current
    
    # Restore dependencies
    cd $APP_DIR/current
    source venv/bin/activate
    pip install -r requirements.txt
    
    # Start application
    log "Starting application..."
    sudo systemctl start python-app
    
    # Health check
    sleep 5
    if curl -f http://localhost:8080/health; then
        log "✅ Rollback successful! Application is healthy"
        # Clean up pre-rollback backup
        rm -rf $BACKUP_DIR/pre_rollback_$timestamp
    else
        log "❌ Rollback failed! Application unhealthy"
        # Restore pre-rollback state
        rm -rf $APP_DIR/current
        cp -r $BACKUP_DIR/pre_rollback_$timestamp $APP_DIR/current
        sudo systemctl restart python-app
        exit 1
    fi
}

# Main execution
case "$1" in
    list)
        list_backups
        ;;
    rollback)
        rollback_to_version "$2"
        ;;
    *)
        echo "Usage: $0 {list|rollback VERSION}"
        exit 1
        ;;
esac
```

**Usage:**
```bash
# List available backups
./rollback.sh list

# Rollback to specific version
./rollback.sh rollback 20241219_143022
```

### Database Backup & Recovery

**Automated Database Backup**
```bash
#!/bin/bash
# db-backup.sh

DB_NAME="appdb"
DB_USER="appuser"
BACKUP_DIR="/var/backups/database"
RETENTION_DAYS=7

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup filename with timestamp
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_$(date +%Y%m%d_%H%M%S).sql.gz"

# Perform backup
pg_dump -U $DB_USER $DB_NAME | gzip > $BACKUP_FILE

# Verify backup
if [ -f "$BACKUP_FILE" ]; then
    echo "✅ Database backup created: $BACKUP_FILE"
    
    # Upload to S3
    aws s3 cp $BACKUP_FILE s3://my-backups/database/
else
    echo "❌ Database backup failed"
    exit 1
fi

# Cleanup old backups
find $BACKUP_DIR -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

# Log
echo "$(date): Backup completed - $BACKUP_FILE" >> /var/log/db-backup.log
```

**Database Restore**
```bash
#!/bin/bash
# db-restore.sh

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup-file>"
    exit 1
fi

# Stop application
sudo systemctl stop python-app

# Restore database
gunzip < $BACKUP_FILE | psql -U appuser appdb

# Start application
sudo systemctl start python-app

echo "✅ Database restored from $BACKUP_FILE"
```

**Cron Job for Automated Backups**
```bash
# Edit crontab
crontab -e

# Add backup jobs
# Daily backup at 2 AM
0 2 * * * /usr/local/bin/db-backup.sh

# Application backup every 6 hours
0 */6 * * * /usr/local/bin/app-backup.sh
```

### Disaster Recovery Plan

**Backup Strategy:**
```
1. Code: GitHub repository (always up-to-date)
2. Database: Daily backups to S3 (retained for 30 days)
3. Application State: Hourly snapshots (retained for 7 days)
4. Configuration: Version controlled in Git
5. Secrets: AWS Secrets Manager
```

**Recovery Time Objectives (RTO):**
- Critical: 15 minutes
- High: 1 hour
- Medium: 4 hours
- Low: 24 hours

**Recovery Point Objectives (RPO):**
- Database: 1 hour (hourly backups)
- Application: 15 minutes (continuous deployment)
- Configuration: Real-time (Git)

### GitHub Actions - Rollback Workflow

```yaml
name: Emergency Rollback

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to rollback to (commit SHA or tag)'
        required: true
        type: string

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout specific version
        uses: actions/checkout@v4
        with:
          ref: ${{ inputs.version }}
      
      - name: Rollback application
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd /var/www/app
            
            # Checkout specific version
            git fetch --all
            git checkout ${{ inputs.version }}
            
            # Reinstall dependencies
            source venv/bin/activate
            pip install -r requirements.txt
            
            # Restart application
            sudo systemctl restart python-app
            
            # Verify
            sleep 5
            curl -f http://localhost:8080/health || exit 1
      
      - name: Notify team
        if: always()
        run: |
          if [ ${{ job.status }} == 'success' ]; then
            echo "✅ Rollback to ${{ inputs.version }} successful"
          else
            echo "❌ Rollback to ${{ inputs.version }} failed"
          fi
```

---

## 13. Best Practices {#best-practices}

### Development Workflow

```
Feature Branch → PR → Code Review → Tests → Merge → Deploy
```

**Branch Strategy:**
```
main (production)
├── develop (staging)
├── feature/user-auth
├── feature/payment-integration
├── hotfix/security-patch
└── release/v1.2.0
```

### Code Review Checklist

- [ ] Code follows style guide
- [ ] Tests are included and passing
- [ ] Documentation is updated
- [ ] No security vulnerabilities
- [ ] Performance impact considered
- [ ] Error handling is comprehensive
- [ ] Logging is appropriate
- [ ] Configuration is externalized
- [ ] Dependencies are up-to-date
- [ ] Backwards compatibility maintained

### CI/CD Best Practices

**1. Fast Feedback**
- Keep build times under 10 minutes
- Run tests in parallel
- Cache dependencies
- Fail fast on errors

**2. Immutable Builds**
- Build once, deploy many times
- Use artifacts/containers
- Version everything
- Tag releases

**3. Environment Parity**
- Dev/Staging/Prod should be similar
- Use Infrastructure as Code
- Consistent configurations
- Same deployment process

**4. Security First**
- Scan for vulnerabilities
- Rotate secrets regularly
- Least privilege access
- Audit logs

**5. Monitoring & Observability**
- Log everything important
- Monitor key metrics
- Set up alerts
- Regular health checks

### Performance Optimization

**Python Application:**
```python
# Use Gunicorn with multiple workers
gunicorn -w 4 -b 0.0.0.0:8080 app:app

# Enable caching
from flask_caching import Cache
cache = Cache(config={'CACHE_TYPE': 'redis'})

# Connection pooling
from sqlalchemy.pool import QueuePool
engine = create_engine(
    'postgresql://...',
    poolclass=QueuePool,
    pool_size=20,
    max_overflow=0
)
```

**Go Application:**
```go
// Set GOMAXPROCS
runtime.GOMAXPROCS(runtime.NumCPU())

// Use connection pooling
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(25)
db.SetConnMaxLifetime(5 * time.Minute)

// Enable HTTP/2
server := &http.Server{
    Addr:         ":8080",
    ReadTimeout:  15 * time.Second,
    WriteTimeout: 15 * time.Second,
    IdleTimeout:  60 * time.Second,
}
```

### Configuration Management

**config.yaml**
```yaml
# Environment-specific configurations
environments:
  development:
    database:
      host: localhost
      port: 5432
      pool_size: 10
    redis:
      host: localhost
      port: 6379
    log_level: debug
    
  staging:
    database:
      host: staging-db.example.com
      port: 5432
      pool_size: 20
    redis:
      host: staging-redis.example.com
      port: 6379
    log_level: info
    
  production:
    database:
      host: prod-db.example.com
      port: 5432
      pool_size: 50
    redis:
      host: prod-redis.example.com
      port: 6379
    log_level: warning
```

---

## 14. Troubleshooting {#troubleshooting}

### Common Issues & Solutions

#### 1. Deployment Fails - Permission Denied

**Problem:**
```
Permission denied (publickey)
```

**Solution:**
```bash
# Check SSH key permissions
chmod 600 ~/.ssh/id_rsa

# Verify SSH connection
ssh -i ~/.ssh/id_rsa ubuntu@EC2_IP

# Add SSH key to ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa

# Test GitHub Actions SSH
ssh -vvv -i ~/.ssh/id_rsa ubuntu@EC2_IP
```

#### 2. Application Won't Start After Deployment

**Problem:**
```
systemctl status app shows "failed"
```

**Solution:**
```bash
# Check service logs
sudo journalctl -u python-app -n 50 --no-pager

# Check application logs
tail -f /var/log/app/error.log

# Verify file permissions
ls -la /var/www/app
sudo chown -R ubuntu:ubuntu /var/www/app

# Check if port is available
sudo netstat -tulpn | grep 8080
sudo lsof -i :8080

# Kill process using the port
sudo kill -9 $(sudo lsof -t -i:8080)

# Restart service
sudo systemctl restart python-app
```

#### 3. Database Connection Failures

**Problem:**
```
OperationalError: could not connect to server
```

**Solution:**
```bash
# Check database is running
sudo systemctl status postgresql

# Test connection
psql -h localhost -U appuser -d appdb

# Check pg_hba.conf
sudo nano /etc/postgresql/14/main/pg_hba.conf

# Add this line if needed
host    all             all             0.0.0.0/0               md5

# Restart PostgreSQL
sudo systemctl restart postgresql

# Check firewall
sudo ufw status
sudo ufw allow 5432/tcp
```

#### 4. Out of Memory Errors

**Problem:**
```
Cannot allocate memory
```

**Solution:**
```bash
# Check memory usage
free -h
top

# Add swap space
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Adjust service memory limits
sudo systemctl edit python-app

[Service]
MemoryLimit=1G
MemoryHigh=800M

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart python-app
```

#### 5. GitHub Actions Timeout

**Problem:**
```
The job running on runner has exceeded the maximum execution time of 360 minutes.
```

**Solution:**
```yaml
# Add timeout to workflow
jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 15  # Adjust as needed
    
    steps:
      - name: Deploy with timeout
        timeout-minutes: 10
        run: |
          # Your deployment commands
```

#### 6. Nginx 502 Bad Gateway

**Problem:**
```
502 Bad Gateway error
```

**Solution:**
```bash
# Check if application is running
curl http://localhost:8080/health

# Check Nginx error log
sudo tail -f /var/log/nginx/error.log

# Verify proxy_pass configuration
sudo nginx -t

# Check if SELinux is blocking (CentOS/RHEL)
sudo setsebool -P httpd_can_network_connect 1

# Restart services
sudo systemctl restart python-app
sudo systemctl restart nginx
```

#### 7. Git Pull Fails During Deployment

**Problem:**
```
error: Your local changes would be overwritten by merge
```

**Solution:**
```bash
# On EC2 instance
cd /var/www/app

# Option 1: Stash changes
git stash
git pull origin main

# Option 2: Force reset
git fetch --all
git reset --hard origin/main

# Option 3: Clean working directory
git clean -fd
git pull origin main
```

#### 8. Environment Variables Not Loading

**Problem:**
```
Configuration value is None or missing
```

**Solution:**
```bash
# Verify .env file exists and has correct permissions
ls -la /var/www/app/.env
chmod 600 /var/www/app/.env

# Check systemd service configuration
systemctl cat python-app

# Ensure EnvironmentFile is specified
[Service]
EnvironmentFile=/var/www/app/.env

# Reload systemd
sudo systemctl daemon-reload
sudo systemctl restart python-app

# Debug: Print environment variables
sudo systemctl show python-app --property=Environment
```

### Debugging Tools & Commands

**System Diagnostics:**
```bash
# CPU usage
top
htop
mpstat 1

# Memory usage
free -h
vmstat 1
cat /proc/meminfo

# Disk usage
df -h
du -sh /var/www/app/*
iostat

# Network
netstat -tulpn
ss -tulpn
iftop

# Process monitoring
ps aux | grep python
pstree -p

# System logs
sudo journalctl -xe
sudo tail -f /var/log/syslog
dmesg | tail
```

**Application Diagnostics:**
```bash
# Check application health
curl -v http://localhost:8080/health

# Test with increased verbosity
curl -v --trace-ascii debug.txt http://localhost:8080/api/test

# Monitor requests
sudo tcpdump -i any -s 0 -A 'tcp port 8080'

# Check application logs
tail -f /var/log/app/*.log | grep ERROR

# Python-specific debugging
# Add to your code:
import pdb; pdb.set_trace()

# Go-specific debugging
# Use delve debugger
dlv debug ./cmd/main.go
```

---

## 15. Cost Optimization {#cost-optimization}

### AWS Cost Management

**Right-Sizing EC2 Instances:**

| Environment | Instance Type | vCPU | Memory | Cost/Month* |
|-------------|--------------|------|--------|-------------|
| Development | t3.micro | 2 | 1 GB | $7.50 |
| Staging | t3.small | 2 | 2 GB | $15.00 |
| Production | t3.medium | 2 | 4 GB | $30.00 |
| Production (HA) | t3.large | 2 | 8 GB | $60.00 |

*Approximate costs in US East region

**Optimization Strategies:**

```bash
# 1. Use Reserved Instances (up to 72% savings)
# Purchase 1-year or 3-year reservations for production

# 2. Use Spot Instances for non-critical workloads
# Configure in terraform or AWS Console

# 3. Auto Scaling
# Scale down during off-hours
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name my-asg \
  --scheduled-action-name scale-down-evening \
  --recurrence "0 19 * * *" \
  --desired-capacity 1

# 4. Stop instances during non-business hours (dev/staging)
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# 5. Use Elastic IPs only when needed ($3.65/month if unattached)
# Release unused Elastic IPs

# 6. Enable CloudWatch detailed monitoring only for production
# Basic monitoring is free, detailed costs $2.10/month per instance

# 7. Optimize storage
# Use gp3 instead of gp2 (20% cheaper)
# Delete old snapshots and unused volumes

# 8. Set up billing alerts
aws cloudwatch put-metric-alarm \
  --alarm-name billing-alarm \
  --alarm-description "Alert when bill exceeds $100" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 21600 \
  --evaluation-periods 1 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold
```

### CI/CD Cost Optimization

**GitHub Actions Minutes:**
```
Free Tier:
- Public repositories: Unlimited
- Private repositories: 2,000 minutes/month

Optimization:
1. Cache dependencies
2. Use matrix builds efficiently
3. Skip unnecessary workflows
4. Use self-hosted runners for heavy workloads
```

**Self-Hosted Runner Setup:**
```bash
# On EC2 instance
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Configure
./config.sh --url https://github.com/YOUR_ORG/YOUR_REPO --token YOUR_TOKEN

# Install as service
sudo ./svc.sh install
sudo ./svc.sh start
```

**Jenkins Cost Optimization:**
```
1. Use EC2 spot instances for Jenkins agents
2. Implement dynamic agent provisioning
3. Clean up old builds automatically
4. Use pipeline stages efficiently
5. Cache Docker layers and dependencies
```

### Storage Cost Optimization

**S3 Lifecycle Policies:**
```json
{
  "Rules": [
    {
      "Id": "ArchiveOldBackups",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 365
      }
    }
  ]
}
```

**Log Retention Policies:**
```bash
# Automatically delete old logs
find /var/log/app -name "*.log" -mtime +30 -delete

# Compress old logs
find /var/log/app -name "*.log" -mtime +7 -exec gzip {} \;

# Setup logrotate
sudo nano /etc/logrotate.d/app

/var/log/app/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 ubuntu ubuntu
    sharedscripts
    postrotate
        systemctl reload python-app > /dev/null
    endscript
}
```

---

## 16. Comparison: GitHub Actions vs Jenkins {#comparison}

### Detailed Feature Comparison

| Feature | GitHub Actions | Jenkins |
|---------|---------------|---------|
| **Setup Complexity** | ⭐⭐⭐⭐⭐ Very Easy | ⭐⭐⭐ Moderate |
| **Maintenance** | ⭐⭐⭐⭐⭐ Zero (Managed) | ⭐⭐ High (Self-managed) |
| **Cost (Small Team)** | ⭐⭐⭐⭐ Free tier generous | ⭐⭐⭐⭐⭐ Free (self-hosted) |
| **Integration** | ⭐⭐⭐⭐⭐ Native GitHub | ⭐⭐⭐⭐ Extensive plugins |
| **Learning Curve** | ⭐⭐⭐⭐ Easy (YAML) | ⭐⭐⭐ Moderate (Groovy) |
| **Scalability** | ⭐⭐⭐⭐ Auto-scaling | ⭐⭐⭐⭐⭐ Highly customizable |
| **Plugin Ecosystem** | ⭐⭐⭐ Growing | ⭐⭐⭐⭐⭐ Extensive |
| **Enterprise Features** | ⭐⭐⭐ Good | ⭐⭐⭐⭐⭐ Excellent |
| **Build Speed** | ⭐⭐⭐⭐ Fast | ⭐⭐⭐⭐ Fast (optimizable) |
| **Security** | ⭐⭐⭐⭐⭐ Managed | ⭐⭐⭐⭐ Self-managed |

### When to Use GitHub Actions

✅ **Perfect For:**
- Startups and small teams
- Projects hosted on GitHub
- Quick setup requirements
- Teams without DevOps expertise
- Public open-source projects
- Microservices architecture
- Cloud-native applications

**Advantages:**
- Zero infrastructure management
- Seamless GitHub integration
- Easy YAML configuration
- Built-in security scanning
- Generous free tier
- Automatic updates
- Matrix builds for multiple platforms

**Limitations:**
- 6-hour job timeout
- Limited to GitHub repositories
- Less customizable than Jenkins
- Vendor lock-in to GitHub
- Pricing can increase with scale

### When to Use Jenkins

✅ **Perfect For:**
- Enterprise organizations
- Complex CI/CD pipelines
- Multi-platform deployments
- Heavy customization needs
- On-premises requirements
- Mixed repository sources (GitHub, GitLab, Bitbucket)
- Legacy system integration

**Advantages:**
- Complete control and customization
- 1,800+ plugins available
- Self-hosted (no vendor lock-in)
- No time limits on jobs
- Advanced pipeline features
- Multi-branch pipelines
- Blue Ocean modern UI

**Limitations:**
- Requires infrastructure management
- Steeper learning curve
- Security updates needed
- Plugin compatibility issues
- Resource intensive

### Hybrid Approach

**Best of Both Worlds:**
```yaml
# Use GitHub Actions for:
- Code quality checks
- Unit testing
- Security scanning
- Pull request validation

# Use Jenkins for:
- Production deployments
- Long-running integration tests
- Complex multi-stage pipelines
- Legacy system integration
```

**Example Hybrid Workflow:**
```yaml
# .github/workflows/pr-checks.yml
name: PR Checks
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: pytest
      
      - name: Trigger Jenkins Deploy (if merged)
        if: github.event.pull_request.merged == true
        run: |
          curl -X POST https://jenkins.example.com/job/deploy/build \
            --user ${{ secrets.JENKINS_USER }}:${{ secrets.JENKINS_TOKEN }}
```

### Cost Comparison Example

**Small Team (5 developers, 1 application):**

**GitHub Actions:**
```
Private repo: 2,000 minutes/month (free)
Additional: ~500 minutes at $0.008/min = $4/month
Storage: Minimal
Total: ~$5-10/month
```

**Jenkins:**
```
EC2 t3.small: $15/month
Storage: $5/month
Maintenance time: 5 hours/month × $50/hr = $250
Total: ~$270/month (including labor)
```

**Large Team (50+ developers, 20+ applications):**

**GitHub Actions:**
```
Enterprise: $21/user/month × 50 = $1,050/month
Additional minutes: ~$500/month
Total: ~$1,550/month
```

**Jenkins:**
```
EC2 t3.xlarge: $150/month
Build agents (5 × t3.medium): $150/month
Storage: $50/month
Maintenance: Full-time DevOps engineer
Total: ~$350/month + DevOps salary
```

### Migration Path

**From Jenkins to GitHub Actions:**
```bash
# 1. Analyze existing Jenkins pipelines
# 2. Convert Jenkinsfile to YAML
# 3. Migrate secrets to GitHub Secrets
# 4. Test in parallel
# 5. Gradual cutover

# Helper tool
npm install -g github-actions-converter
github-actions-converter Jenkinsfile > .github/workflows/ci.yml
```

**From GitHub Actions to Jenkins:**
```groovy
// 1. Setup Jenkins server
// 2. Install required plugins
// 3. Create Jenkinsfile
// 4. Configure webhooks
// 5. Migrate secrets to Jenkins credentials

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                // Convert YAML steps to Groovy
                sh 'make build'
            }
        }
    }
}
```

---

## 17. Advanced Topics

### Docker Integration

**Dockerfile - Python**
```dockerfile
FROM python:3.10-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080

CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8080", "app:app"]
```

**Dockerfile - Go**
```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder

WORKDIR /build

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app ./cmd/main.go

# Runtime stage
FROM alpine:latest

RUN apk --no-cache add ca-certificates
WORKDIR /root/

COPY --from=builder /build/app .

EXPOSE 8080

CMD ["./app"]
```

**GitHub Actions - Docker Build & Push**
```yaml
name: Docker Build & Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            myapp/app:latest
            myapp/app:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            docker pull myapp/app:latest
            docker stop myapp || true
            docker rm myapp || true
            docker run -d \
              --name myapp \
              -p 8080:8080 \
              --restart unless-stopped \
              myapp/app:latest
```

### Kubernetes Deployment

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp/app:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-host
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
```

### Infrastructure as Code (Terraform)

**main.tf**
```hcl
provider "aws" {
  region = "us-east-1"
}

# EC2 Instance
resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"  # Ubuntu 22.04
  instance_type = "t3.medium"
  key_name      = aws_key_pair.deployer.key_name

  vpc_security_group_ids = [aws_security_group.app_sg.id]

  tags = {
    Name = "AppServer"
    Environment = "Production"
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update
              apt-get install -y python3 python3-pip nginx
              EOF
}

# Security Group
resource "aws_security_group" "app_sg" {
  name        = "app-security-group"
  description = "Security group for application server"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["YOUR_IP/32"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Key Pair
resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = file("~/.ssh/id_rsa.pub")
}

# Elastic IP
resource "aws_eip" "app_eip" {
  instance = aws_instance.app_server.id
  domain   = "vpc"
}

output "instance_ip" {
  value = aws_eip.app_eip.public_ip
}
```

---

## 18. Conclusion

This comprehensive CI/CD documentation provides production-ready configurations for deploying Python and Golang applications using GitHub Actions and Jenkins on AWS EC2. The guide covers everything from basic setup to advanced topics like blue-green deployments, monitoring, security, and disaster recovery.

### Key Takeaways

**For Startups & Small Teams:**
- Start with GitHub Actions for simplicity
- Use single EC2 instance with proper monitoring
- Implement automated backups from day one
- Focus on quick iterations and deployment velocity

**For Growing Teams:**
- Consider Jenkins for complex workflows
- Implement staging environments
- Add comprehensive monitoring and alerting
- Establish proper code review processes

**For Enterprise:**
- Use Jenkins for complete control
- Implement multi-region deployments
- Advanced security and compliance measures
- Full disaster recovery capabilities

### Next Steps

1. **Choose Your Tool**: GitHub Actions (easy start) or Jenkins (enterprise control)
2. **Set Up Infrastructure**: Provision EC2, configure security groups
3. **Implement Basic Pipeline**: Start with simple build-test-deploy
4. **Add Quality Gates**: Linting, testing, security scans
5. **Implement Monitoring**: Logs, metrics, alerts
6. **Establish Backup Strategy**: Code, data, configurations
7. **Document Everything**: Keep this documentation updated
8. **Iterate and Improve**: Continuously optimize your pipeline

### Additional Resources

**Official Documentation:**
- GitHub Actions: https://docs.github.com/actions
- Jenkins: https://www.jenkins.io/doc/
- AWS EC2: https://docs.aws.amazon.com/ec2/
- Docker: https://docs.docker.com/
- Kubernetes: https://kubernetes.io/docs/

**Community Resources:**
- GitHub Actions Marketplace: https://github.com/marketplace
- Jenkins Plugins: https://plugins.jenkins.io/
- DevOps Subreddit: https://reddit.com/r/devops
- Stack Overflow: Tag your questions appropriately

**Best Practice Guides:**
- 12-Factor App: https://12factor.net/
- Google SRE Book: https://sre.google/books/
- DevOps Handbook: https://itrevolution.com/

### Support & Feedback

If you encounter issues or have suggestions:
1. Check the troubleshooting section
2. Review AWS CloudWatch logs
3. Consult community forums
4. Open GitHub issues for tool-specific problems

---

**Document Version:** 2.0  
**Last Updated:** December 2024  
**Authors:** DevOps Team  
**License:** MIT

---

### Quick Reference Commands

```bash
# GitHub Actions
gh workflow list
gh workflow run ci.yml
gh run list
gh run view <run-id>

# Jenkins
java -jar jenkins-cli.jar -s http://localhost:8080/ build job-name
java -jar jenkins-cli.jar -s http://localhost:8080/ list-jobs

# AWS EC2
aws ec2 describe-instances
aws ec2 start-instances --instance-ids i-xxxxx
aws ec2 stop-instances --instance-ids i-xxxxx

# System Management
sudo systemctl status app
sudo systemctl restart app
sudo journalctl -u app -f
tail -f /var/log/app/app.log

# Docker
docker ps
docker logs -f container-name
docker exec -it container-name /bin/bash
docker system prune -a

# Git
git log --oneline -10
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin v1.0.0
```

---

**End of Documentation**

For updates and contributions, visit our GitHub repository or contact the DevOps team.('Linting') {
                    steps {
                        sh '''
                            . venv/bin/activate
                            flake8 . --max-line-length=100 --exclude=venv > flake8-report.txt || true
                        '''
                    }
                }
                stage('Format Check') {
                    steps {
                        sh '''
                            . venv/bin/activate
                            black --check . || true
                        '''
                    }
                }
                stage('Security Scan') {
                    steps {
                        sh '''
                            . venv/bin/activate
                            pip install bandit
                            bandit -r . -f json -o bandit-report.json || true
                        '''
                    }
                }
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    . venv/bin/activate
                    pytest tests/ -v --cov=src --cov-report=xml --cov-report=html
                '''
            }
            post {
                always {
                    junit 'test-results.xml'
                    publishHTML([
                        reportDir: 'htmlcov',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
        
        stage('Build') {
            steps {
                echo 'Creating deployment package...'
                sh '''
                    mkdir -p dist
                    cp -r src dist/
                    cp requirements.txt dist/
                '''
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} "
                            set -e
                            cd ${APP_DIR}
                            
                            # Backup current version
                            if [ -d current ]; then
                                timestamp=\$(date +%Y%m%d_%H%M%S)
                                mv current backup_\$timestamp
                            fi
                            
                            # Pull latest code
                            git pull origin main
                            
                            # Setup virtual environment
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install --upgrade pip
                            pip install -r requirements.txt
                            
                            # Restart application
                            sudo systemctl restart python-app
                            
                            # Health check
                            sleep 5
                            curl -f http://localhost:8080/health || exit 1
                            
                            echo 'Deployment successful!'
                        "
                    '''
                }
            }
        }
        
        stage('Smoke Tests') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    sleep 10
                    curl -f http://${EC2_HOST}/health
                    curl -f http://${EC2_HOST}/api/status
                '''
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline completed successfully!'
            // Send notification (Slack, Email, etc.)
        }
        failure {
            echo '❌ Pipeline failed!'
            // Rollback logic
            sshagent(credentials: ['ec2-ssh-key']) {
                sh '''
                    ssh ubuntu@${EC2_HOST} "
                        cd ${APP_DIR}
                        if [ -d backup_* ]; then
                            latest_backup=\$(ls -td backup_* | head -1)
                            mv \$latest_backup current
                            sudo systemctl restart python-app
                        fi
                    "
                '''
            }
        }
        always {
            cleanWs()
        }
    }
}
```

### Golang Application - Jenkinsfile

```groovy
pipeline {
    agent any
    
    environment {
        APP_NAME = 'go-app'
        APP_DIR = '/home/ubuntu/app'
        GO_VERSION = '1.22'
    }
    
    tools {
        go 'go-1.22'
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 20, unit: 'MINUTES')
        timestamps()
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup') {
            steps {
                sh '''
                    go version
                    go mod download
                    go mod verify
                '''
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('Format Check') {
                    steps {
                        sh '''
                            if [ "$(gofmt -s -l . | wc -l)" -gt 0 ]; then
                                echo "Code not formatted"
                                gofmt -s -l .
                                exit 1
                            fi
                        '''
                    }
                }
                stage
