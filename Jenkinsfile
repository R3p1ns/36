pipeline {
    agent any
    environment {
        // AWS Credentials
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        EKS_CLUSTER_NAME     = 'lan-cuoi'
        AWS_REGION           = 'us-east-1'
        DOCKER_IMAGE         = "hieupro7410/flask-app"
        DOCKER_TAG          = "build-${env.BUILD_NUMBER}" // Thay thế ${env.BUILD_ID} bằng BUILD_NUMBER
    }
    stages {
        /* Stage 1: Build Docker Image */
        stage('Build Docker Image') {
            steps {
                script {
                    // Build với cả repository và tag rõ ràng
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                }
            }
        }

        /* Stage 2: Push to Docker Hub */
        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-cred') {
                        // Push cả image và tag
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                        // Optionally push as latest
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push('latest')
                    }
                }
            }
        }

        /* Stage 3: Deploy to EKS */
        stage('Deploy to EKS') {
            steps {
                script {
                    sh """
                    aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}
                    sed -i 's|<DOCKER_IMAGE>|${DOCKER_IMAGE}:${DOCKER_TAG}|g' k8s-deployment.yaml
                    kubectl apply -f k8s-deployment.yaml
                    kubectl rollout status deployment/flask-app --timeout=2m
                    """
                }
            }
        }

        /* Stage 4: End-to-End Test */
        stage('E2E Test') {
            steps {
                script {
                    def LB_URL = sh(
                        returnStdout: true,
                        script: "kubectl get svc flask-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'"
                    ).trim()

                    sh """
                    curl -s --retry 5 --retry-delay 10 "http://${LB_URL}" | grep "Hello DevOps CI/CD Pipeline!"
                    """
                }
            }
        }
    }
}