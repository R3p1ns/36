pipeline {
    agent any
    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("flask-app:${env.BUILD_ID}")
                }
            }
        }
        stage('Run Container') {
            steps {
                script {
                    docker.image("flask-app:${env.BUILD_ID}").run('-p 5000:5000 -d')
                }
            }
        }
    }
}
