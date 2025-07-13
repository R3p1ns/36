pipeline {
    agent any
    environment {
        // AWS Credentials
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        EKS_CLUSTER_NAME     = 'lan-cuoi'  // Thay bằng tên EKS cluster của bạn
        AWS_REGION           = 'us-east-1'  // Thay bằng region của bạn
        DOCKER_IMAGE         = 'hieupro7410/flask-app' // Bỏ tag BUILD_ID tạm thời
    }
    stages {
        /* Stage 1: Build Docker Image */
        stage('Build Docker Image') {
            steps {
                script {
                    // Sử dụng tag đơn giản
                    docker.build("${DOCKER_IMAGE}")
                }
            }
        }

        /* Stage 2: Run Unit Tests */
        stage('Unit Tests') {
    steps {
        sh '''
        python3 -m venv venv
        . venv/bin/activate
        pip install pytest pytest-cov
        python -m pytest test_app.py --cov=app --cov-report=xml || true
        '''
    }
}

        /* Stage 3: Push to Docker Hub */
        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-cred') {
                        // Thêm tag hợp lệ
                        docker.image("${DOCKER_IMAGE}").push('latest')
                    }
                }
            }
        }

        /* Stage 4: Deploy to EKS */
        stage('Deploy to EKS') {
            steps {
                script {
                    sh """
                    aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}
                    sed -i 's|<DOCKER_IMAGE>|${DOCKER_IMAGE}:latest|g' k8s-deployment.yaml
                    kubectl apply -f k8s-deployment.yaml
                    kubectl rollout status deployment/flask-app --timeout=2m
                    """
                }
            }
        }
    }
    post {
        always {
            // Thay thế Slack notification bằng email hoặc ghi log
            echo "Pipeline completed - Status: ${currentBuild.result}"
            archiveArtifacts artifacts: '**/target/*.jar, **/test-reports/*.xml', allowEmptyArchive: true
        }
    }
}