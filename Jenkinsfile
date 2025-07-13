pipeline {
    agent {
        docker {
            image 'docker:24.0.7'
            args '-v /var/run/docker.sock:/var/run/docker.sock --group-add $(stat -c %g /var/run/docker.sock)'
        }
    }
    environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        EKS_CLUSTER_NAME     = 'lan-cuoi'
        AWS_REGION           = 'us-east-1'
        DOCKER_IMAGE         = 'hieupro7410/flask-app'
    }
    stages {
        /* Stage 1: Build Docker Image */
        stage('Build') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}")
                }
            }
        }

        /* Stage 2: Push to Docker Hub */
        stage('Push') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-cred') {
                        docker.image("${DOCKER_IMAGE}").push('latest')
                    }
                }
            }
        }

        /* Stage 3: Deploy to EKS */
        stage('Deploy') {
            steps {
                script {
                    sh """
                    # Install required tools
                    apk add --no-cache aws-cli kubectl

                    # Configure AWS CLI
                    aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
                    aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
                    aws configure set region ${AWS_REGION}

                    # Update kubeconfig
                    aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}

                    # Deploy to EKS
                    sed -i "s|<DOCKER_IMAGE>|${DOCKER_IMAGE}:latest|g" k8s-deployment.yaml
                    kubectl apply -f k8s-deployment.yaml
                    kubectl rollout status deployment/flask-app --timeout=2m
                    """
                }
            }
        }
    }
    post {
        always {
            echo "Deployment completed with status: ${currentBuild.currentResult}"
        }
        success {
            echo "✅ Successfully deployed ${DOCKER_IMAGE} to EKS cluster ${EKS_CLUSTER_NAME}"
        }
        failure {
            echo "❌ Deployment failed! Check logs for details"
        }
    }
}