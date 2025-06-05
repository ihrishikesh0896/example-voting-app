pipeline {
    agent any

    environment {
        DOCKER_COMPOSE_VERSION = '2.24.6'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/ihrishikesh0896/example-voting-app.git'
            }
        }
        stage('Install Docker Compose') {
            steps {
                sh '''
                  sudo curl -SL "https://github.com/docker/compose/releases/download/v${DOCKER_COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
                  sudo chmod +x /usr/local/bin/docker-compose
                  docker-compose --version
                '''
            }
        }
        stage('Build & Deploy') {
            steps {
                sh '''
                  docker compose down || true
                  docker compose up -d --build
                '''
            }
        }
    }
    post {
        always {
            sh 'docker compose ps'
        }
    }
}
