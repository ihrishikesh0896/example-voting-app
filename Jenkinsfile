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
