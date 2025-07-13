pipeline {
    agent any
    }
    environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        EKS_CLUSTER_NAME     = 'lan-cuoi'
        AWS_REGION           = 'us-east-1'
        DOCKER_IMAGE         = 'hieupro7410/flask-app'
    }
    stages {
        /* Stage 1: Verify Docker Access */
        stage('Check Docker') {
            steps {
                sh 'docker system info'
            }
        }

        /* Stage 2: Unit Tests */
        stage('Unit Tests') {
            agent {
                docker {
                    image 'python:3.9-slim'
                    reuseNode true
                }
            }
            steps {
                sh '''
                python -m venv /tmp/venv
                source /tmp/venv/bin/activate
                pip install pytest pytest-cov
                python -m pytest test_app.py --cov=app --cov-report=xml || true
                '''
            }
        }

        /* Stage 3: Build & Push */
        stage('Build and Push') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}")
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
                    sh '''
                    apk add --no-cache aws-cli kubectl
                    aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}
                    sed -i "s|<DOCKER_IMAGE>|${DOCKER_IMAGE}:latest|g" k8s-deployment.yaml
                    kubectl apply -f k8s-deployment.yaml --timeout=2m
                    '''
                }
            }
        }
    }
}