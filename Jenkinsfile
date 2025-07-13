pipeline {
    agent any
    environment {
        EKS_CLUSTER_NAME     = 'lan-cuoi'
        AWS_REGION           = 'us-east-1'
        DOCKER_IMAGE         = "hieupro7410/flask-app"
        DOCKER_TAG          = "build-${env.BUILD_NUMBER}"
    }
    stages {
        /* Stage 1: Build Docker Image */
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                }
            }
        }

        /* Stage 2: Push to Docker Hub */
        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-cred') {
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push('latest')
                    }
                }
            }
        }

        stage('Deploy to EKS') {
    steps {
        script {
            withCredentials([
                string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
            ]) {
                sh """
                # Cấu hình AWS CLI
                aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
                aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
                aws configure set region ${AWS_REGION}

                # Cập nhật kubeconfig với đầy đủ quyền
                aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}

                # Kiểm tra kết nối
                kubectl get nodes

                # Triển khai ứng dụng
                sed -i 's|<DOCKER_IMAGE>|${DOCKER_IMAGE}:${DOCKER_TAG}|g' k8s-deployment.yaml
                kubectl apply -f k8s-deployment.yaml
                kubectl rollout status deployment/flask-app --timeout=2m
                """
            }
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
    post {
        always {
            echo "Pipeline completed - Status: ${currentBuild.currentResult}"
        }
    }
}