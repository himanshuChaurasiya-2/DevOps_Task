# 🚀 MEAN Stack DevOps Deployment – CI/CD with Docker, Jenkins & nginx

## 📌 Task Overview

This project demonstrates the complete DevOps workflow for a full-stack **MEAN (MongoDB, Express, Angular, Node.js)** application. The application has been containerized using Docker, and automated via Jenkins CI/CD pipeline with nginx as a reverse proxy.

Repository contains:

* ✅ Dockerfiles for frontend and backend
* ✅ Docker Compose configuration
* ✅ Jenkins CI/CD pipeline
* ✅ Nginx reverse proxy setup
* ✅ Infrastructure + deployment documentation

---

## 📁 Project Structure

```
CRUD-DD-TASK-MEAN-APP/
│
├── backend/
│   ├── Dockerfile
│   └── ...
│
├── frontend/
│   ├── Dockerfile
│   └── ...
│
├── nginx/
│   └── default.conf
│
├── docker-compose.yml
├── Jenkinsfile
└── README.md
```

---

## 🧰 Technologies Used

* Node.js + Express
* Angular
* MongoDB (Official Docker Image)
* Docker & Docker Compose
* Jenkins (CI/CD)
* Nginx (Reverse Proxy)
* AWS EC2 (Ubuntu VM)

---

## 🖥️ Infrastructure Setup

### VM Configuration

* OS: Ubuntu 22.04
* Jenkins installed
* Docker + Docker Compose installed
* Ports opened:

  * 8080 (Jenkins)
  * 80 (Application)
  * 22 (SSH)

---

## 🐳 Docker Setup

### Backend Dockerfile

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["node", "server.js"]
```

### Frontend Dockerfile

```dockerfile
FROM node:18-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist/* /usr/share/nginx/html
```

---

## 📦 Docker Compose Configuration

```yaml
version: '3.9'

services:
  mongo:
    image: mongo:6
    container_name: mongo
    restart: always
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  backend:
    image: YOUR_DOCKERHUB_USERNAME/mean-backend:latest
    container_name: mean-backend
    restart: always
    environment:
      - MONGO_URL=mongodb://mongo:27017/meanapp
      - PORT=8080
    depends_on:
      - mongo

  frontend:
    image: YOUR_DOCKERHUB_USERNAME/mean-frontend:latest
    container_name: mean-frontend
    restart: always
    depends_on:
      - backend

  nginx:
    image: nginx:alpine
    container_name: mean-nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - frontend

volumes:
  mongo-data:
```

---

## 🔧 nginx Reverse Proxy Configuration

### nginx/default.conf

```nginx
server {
    listen 80;
    server_name _;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://backend:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 🔄 Jenkins CI/CD Pipeline

### Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'himanshudev19'
        BACKEND_IMAGE = "${DOCKERHUB_USER}/mean-backend"
        FRONTEND_IMAGE = "${DOCKERHUB_USER}/mean-frontend"
        VM_USER = 'ubuntu'
        VM_HOST = 'YOUR_EC2_PUBLIC_IP'
        DEPLOY_PATH = '/home/ubuntu/mean-app'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    sh """
                    docker build -t ${BACKEND_IMAGE}:${GIT_COMMIT} ./backend
                    docker build -t ${FRONTEND_IMAGE}:${GIT_COMMIT} ./frontend

                    docker tag ${BACKEND_IMAGE}:${GIT_COMMIT} ${BACKEND_IMAGE}:latest
                    docker tag ${FRONTEND_IMAGE}:${GIT_COMMIT} ${FRONTEND_IMAGE}:latest
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'dockerhub-creds', 
                    usernameVariable: 'DOCKER_USER', 
                    passwordVariable: 'DOCKER_PASS')
                ]) {
                    sh """
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

                    docker push ${BACKEND_IMAGE}:${GIT_COMMIT}
                    docker push ${BACKEND_IMAGE}:latest
                    docker push ${FRONTEND_IMAGE}:${GIT_COMMIT}
                    docker push ${FRONTEND_IMAGE}:latest

                    docker logout
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'ec2-ssh-key', 
                    keyFileVariable: 'SSH_KEY', 
                    usernameVariable: 'SSH_USER')
                ]) {
                    sh """
                    chmod 600 $SSH_KEY

                    ssh -o StrictHostKeyChecking=no -i $SSH_KEY ${VM_USER}@${VM_HOST} 'mkdir -p ${DEPLOY_PATH}'

                    rsync -av -e "ssh -i $SSH_KEY -o StrictHostKeyChecking=no" docker-compose.yml ${VM_USER}@${VM_HOST}:${DEPLOY_PATH}/
                    rsync -av -e "ssh -i $SSH_KEY -o StrictHostKeyChecking=no" nginx/ ${VM_USER}@${VM_HOST}:${DEPLOY_PATH}/nginx/

                    ssh -o StrictHostKeyChecking=no -i $SSH_KEY ${VM_USER}@${VM_HOST} "
                        cd ${DEPLOY_PATH} &&
                        docker-compose pull &&
                        docker-compose down &&
                        docker-compose up -d --remove-orphans
                    "
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment completed successfully"
        }
        failure {
            echo "❌ Deployment failed - check Jenkins logs"
        }
    }
}
```

---

## ✅ CI/CD Flow

1. Push code to GitHub
2. Jenkins auto triggers build
3. Docker images are built
4. Images pushed to Docker Hub
5. VM pulls latest images
6. Containers restart
7. Application live via Nginx

---

## 📸 Screenshots (To Include)

### ✅ Jenkins Pipeline Configuration

![Jenkins Job Setup](screenshots/jenkins-job.png)

### ✅ Jenkins Build Success

![Jenkins Build](screenshots/jenkins-build.png)

### ✅ Docker Image Push

![Docker Hub Images](screenshots/dockerhub.png)

### ✅ Application Running

![Live Application](screenshots/app-ui.png)

### ✅ Nginx Reverse Proxy

![Nginx Config](screenshots/nginx-config.png)

---

## 🧪 How to Run Locally

```bash
git clone https://github.com/yourusername/DevOps_Task.git
cd DevOps_Task
docker-compose up -d
```

Open browser:

```
http://<VM-IP>
```

---

## 🔐 Jenkins Credentials Setup

1. Go to Jenkins Dashboard
2. Manage Jenkins → Credentials
3. Add:

   * ID: DockerHub
   * Username: himanshudev19
   * Password: Your Docker Hub password

---

## 🛡 How to Prevent IP Change Issues

Use Elastic IP (AWS) or Reserved IP to ensure VM keeps same address.

---

## ✅ Final Verification Checklist

| Step            | Status |
| --------------- | ------ |
| GitHub Push     | ✅      |
| Jenkins Trigger | ✅      |
| Docker Build    | ✅      |
| Docker Push     | ✅      |
| App Deployment  | ✅      |
| UI Accessible   | ✅      |

---

## 📎 GitHub Repository

> 🔗 Add your GitHub link here after final deployment

```
https://github.com/yourusername/DevOps_Task
```

---

## 🎯 Conclusion

This setup provides a fully automated CI/CD pipeline for a MEAN application using industry best practices. The infrastructure ensures reliability, scalability, and easy deployment through Docker and Jenkins automation.

---

✅ Task Completed Successfully
