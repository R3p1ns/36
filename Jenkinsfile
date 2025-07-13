pipeline {
    agent any
    environment {
        // AWS Credentials
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        EKS_CLUSTER_NAME     = 'lan-cuoi'  // Thay bằng tên EKS cluster của bạn
        AWS_REGION           = 'us-east-1'  // Thay bằng region của bạn
        DOCKER_IMAGE         = 'your-dockerhub/flask-app:${env.BUILD_ID}'
    }
    stages {
        /* Stage 1: Build Docker Image */
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}")
                }
            }
        }

        /* Stage 2: Run Unit Tests */
        stage('Unit Tests') {
            steps {
                sh '''
                python -m pip install pytest pytest-cov
                python -m pytest test_app.py --cov=app --cov-report=xml
                '''
            }
            post {
                always {
                    junit 'test-reports/*.xml'  // Xuất báo cáo JUnit
                    cobertura coberturaReportFile: 'coverage.xml'  // Xuất báo cáo coverage
                }
            }
        }

        /* Stage 3: Push to Docker Hub */
        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-cred') {
                        docker.image("${DOCKER_IMAGE}").push()
                    }
                }
            }
        }

        /* Stage 4: Deploy to EKS */
        stage('Deploy to EKS') {
            steps {
                script {
                    // Cập nhật kubeconfig
                    sh """
                    aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}
                    """

                    // Thay thế image name trong file deployment
                    sh "sed -i 's|<DOCKER_IMAGE>|${DOCKER_IMAGE}|g' k8s-deployment.yaml"

                    // Áp dụng cấu hình Kubernetes
                    sh """
                    kubectl apply -f k8s-deployment.yaml
                    kubectl rollout status deployment/flask-app --timeout=2m
                    """
                }
            }
        }

        /* Stage 5: End-to-End Test */
        stage('E2E Test') {
            steps {
                script {
                    // Lấy địa chỉ LoadBalancer
                    def LB_URL = sh(
                        returnStdout: true,
                        script: "kubectl get svc flask-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'"
                    ).trim()

                    // Kiểm tra ứng dụng
                    sh """
                    echo "Testing endpoint: http://${LB_URL}"
                    curl -s --retry 5 --retry-delay 10 "http://${LB_URL}" | grep "Hello DevOps CI/CD Pipeline!"
                    """
                }
            }
        }
    }
    post {
        success {
            slackSend(
                channel: '#devops-alerts',
                message: "✅ Pipeline SUCCESS - Build ${env.BUILD_NUMBER}\nDeployed: ${DOCKER_IMAGE}\nEKS Cluster: ${EKS_CLUSTER_NAME}"
            )
        }
        failure {
            slackSend(
                channel: '#devops-alerts',
                message: "❌ Pipeline FAILED - Build ${env.BUILD_NUMBER}\nXem log tại: ${env.BUILD_URL}"
            )
        }
    }
}