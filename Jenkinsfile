pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = 'reddamsandeep'
        IMAGE_NAME = 'java-cicd-demo'
        IMAGE_TAG = "${BUILD_NUMBER}"
        K8S_DEPLOYMENT = 'java-cicd-deployment'
        K8S_SERVICE = 'java-cicd-service'
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                echo 'Building Java application using Maven...'
                sh 'mvn clean package'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh '''
                    docker build -t $DOCKERHUB_USERNAME/$IMAGE_NAME:$IMAGE_TAG .
                    docker tag $DOCKERHUB_USERNAME/$IMAGE_NAME:$IMAGE_TAG $DOCKERHUB_USERNAME/$IMAGE_NAME:latest
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                echo 'Pushing Docker image to Docker Hub...'

                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        docker push $DOCKERHUB_USERNAME/$IMAGE_NAME:$IMAGE_TAG
                        docker push $DOCKERHUB_USERNAME/$IMAGE_NAME:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying application to Kubernetes...'

                sh '''
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml

                    kubectl set image deployment/$K8S_DEPLOYMENT \
                        java-cicd-container=$DOCKERHUB_USERNAME/$IMAGE_NAME:$IMAGE_TAG

                    kubectl rollout status deployment/$K8S_DEPLOYMENT
                '''
            }
        }

        stage('Expose Application') {
            steps {
                echo 'Starting port-forward for browser access...'

                sh '''
                    export JENKINS_NODE_COOKIE=dontKillMe

                    nohup kubectl port-forward \
                        --address=0.0.0.0 \
                        service/$K8S_SERVICE \
                        30080:8080 \
                        > java-app-port-forward.log 2>&1 &

                    sleep 5

                    echo "Checking port-forward process..."
                    ps -ef | grep "kubectl port-forward" | grep -v grep || true

                    echo "Testing application..."
                    curl http://127.0.0.1:30080/ || true

                    echo "Application URL:"
                    echo "http://kuberops.centralindia.cloudapp.azure.com:30080/"
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                echo 'Verifying Kubernetes deployment...'

                sh '''
                    echo "=== Deployment ==="
                    kubectl get deployments

                    echo "=== Pods ==="
                    kubectl get pods

                    echo "=== Services ==="
                    kubectl get svc

                    echo "=== Application Test ==="
                    curl http://127.0.0.1:30080/ || true
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
            echo 'Application URL: http://kuberops.centralindia.cloudapp.azure.com:30080/'
        }

        failure {
            echo 'CI/CD pipeline failed. Check Jenkins console output.'
        }
    }
}
