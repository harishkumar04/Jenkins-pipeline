pipeline {
    agent {
        label 'docker'
    }
    
    environment{
        DOCKER_IMAGE = "harishkumar09/node-k8s-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {

        stage('Clone Code') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $DOCKER_IMAGE:$IMAGE_TAG .
                '''
            }
        }
        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push $DOCKER_IMAGE:$IMAGE_TAG
                    '''
                }
            }
        }
        stage('Deploy to Kind') {
            steps {
                sh '''
                    sed -i "s|image:.*|image: $DOCKER_IMAGE:$IMAGE_TAG|g" k8s/deployment.yaml
                    kubectl apply -f k8s/
                '''
            }
        }
    }
    post {
        success {
            echo 'Deployment successful'
        }

        failure {
            echo 'Pipeline failed'
        }
    }
}
