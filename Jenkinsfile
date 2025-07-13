pipeline {
    agent {
        docker {
            image 'docker:24.0.7-dind'
            args '--privileged -v /var/run/docker.sock:/var/run/docker.sock'
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
        /* Stage 0: Setup Docker Environment */
        stage('Prepare Docker') {
            steps {
                script {
                    sh '''
                    sudo chmod 777 /var/run/docker.sock
                    docker system info
                    '''
                }
            }
        }


        /* Stage 2: Build Docker Image */
        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t ${DOCKER_IMAGE} .'
                }
            }
        }

        /* Stage 3: Push to Docker Hub */
        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${DOCKER_IMAGE}:latest
                        '''
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
                    aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
                    aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
                    aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}
                    sed -i "s|<DOCKER_IMAGE>|${DOCKER_IMAGE}:latest|g" k8s-deployment.yaml
                    kubectl apply -f k8s-deployment.yaml
                    kubectl rollout status deployment/flask-app --timeout=2m
                    '''
                }
            }
        }
    }
}