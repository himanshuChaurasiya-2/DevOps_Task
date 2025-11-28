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
