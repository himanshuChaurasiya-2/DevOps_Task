pipeline {
    agent any

    environment {
        DOCKERHUB_USER = credentials('dockerhub-creds')
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/YOUR_USER/YOUR_REPO.git'
            }
        }

        stage('Build Backend Image') {
            steps {
                sh "docker build -t ${DOCKERHUB_USER_USR}/mean-backend:latest ./backend"
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh "docker build -t ${DOCKERHUB_USER_USR}/mean-frontend:latest ./frontend"
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh """
                echo "${DOCKERHUB_USER_PSW}" | docker login -u "${DOCKERHUB_USER_USR}" --password-stdin
                docker push ${DOCKERHUB_USER_USR}/mean-backend:latest
                docker push ${DOCKERHUB_USER_USR}/mean-frontend:latest
                """
            }
        }

        stage('Deploy to VM') {
            steps {
                sshagent(['vm-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@YOUR_VM_IP '
                        cd /app &&
                        git pull &&
                        docker-compose pull &&
                        docker-compose up -d
                    '
                    """
                }
            }
        }
    }
}
