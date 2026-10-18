pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '848504403730'
        ECR_REGISTRY = '848504403730.dkr.ecr.ap-south-1.amazonaws.com'
        FRONTEND_REPO = 'three-tier-frontend'
        BACKEND_REPO = 'three-tier-backend'
        FRONTEND_BACKEND_URL = 'http://k8s-threetie-mainlb-e88fc63846-1372557241.ap-south-1.elb.amazonaws.com/api/tasks'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()
                    echo "Building images with tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Build Images') {
            steps {
                sh '''
                    docker build \
                      -t "$ECR_REGISTRY/$BACKEND_REPO:$IMAGE_TAG" \
                      Application-Code/backend

                    docker build \
                      --build-arg REACT_APP_BACKEND_URL="$FRONTEND_BACKEND_URL" \
                      -t "$ECR_REGISTRY/$FRONTEND_REPO:$IMAGE_TAG" \
                      Application-Code/frontend
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                    set -eu
                    aws ecr get-login-password --region "$AWS_REGION" |
                      docker login --username AWS --password-stdin "$ECR_REGISTRY"

                    docker push "$ECR_REGISTRY/$BACKEND_REPO:$IMAGE_TAG"
                    docker push "$ECR_REGISTRY/$FRONTEND_REPO:$IMAGE_TAG"
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout "$ECR_REGISTRY" || true'
        }
    }
}
