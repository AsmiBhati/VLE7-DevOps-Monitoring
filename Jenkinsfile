
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'AsmiBhati/vle7-devops-monitoring'
        EC2_HOST = '98.81.178.30'
        SSH_CREDENTIALS = 'ec2-ssh-key'
        DOCKER_CREDENTIALS = 'dockerhub-creds'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                      -t ${DOCKER_IMAGE}:latest .
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        echo "$DOCKER_TOKEN" | docker login \
                          -u "$DOCKER_USER" --password-stdin

                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        scp -o StrictHostKeyChecking=accept-new \
                          k8s/deployment.yaml \
                          k8s/service.yaml \
                          ubuntu@${EC2_HOST}:/tmp/

                        ssh -o StrictHostKeyChecking=accept-new \
                          ubuntu@${EC2_HOST} \
                          'sudo k3s kubectl apply -f /tmp/deployment.yaml &&
                           sudo k3s kubectl apply -f /tmp/service.yaml'

                        ssh -o StrictHostKeyChecking=accept-new \
                          ubuntu@${EC2_HOST} \
                          'sudo k3s kubectl set image \
                           deployment/vle7-app \
                           vle7-app=${DOCKER_IMAGE}:${BUILD_NUMBER} &&
                           sudo k3s kubectl rollout status \
                           deployment/vle7-app --timeout=120s'
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Application built, pushed and deployed!'
        }

        failure {
            echo 'FAILED: Check the Jenkins console output.'
        }
    }
}
