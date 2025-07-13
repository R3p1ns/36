pipeline {
    agent {
        docker {
            image 'docker:24.0.7-dind'
            args '--privileged -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        // AWS Credentials
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        EKS_CLUSTER_NAME     = 'lan-cuoi'
        AWS_REGION           = 'us-east-1'
        DOCKER_IMAGE         = 'hieupro7410/flask-app'
    }
    stages {
        /* Stage 1: Setup Python Test Environment */
        stage('Prepare Test Environment') {
            agent {
                docker {
                    image 'python:3.9-slim'
                    reuseNode true
                }
            }
            steps {
                sh '''
                pip install pytest pytest-cov
                python -m pytest test_app.py --cov=app --cov-report=xml || true
                '''
            }
        }

        /* Stage 2: Build Docker Image */
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}")
                }
            }
        }

        /* Stage 3: Push to Docker Hub */
        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-cred') {
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
                    apk add --no-cache aws-cli kubectl
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
            echo "Pipeline completed - Status: ${currentBuild.result}"
            archiveArtifacts artifacts: '**/test-reports/*.xml', allowEmptyArchive: true
        }
    }
}