pipeline {
    agent any

    environment {
        DOCKER_COMPOSE_VERSION = '2.24.6'
        IMAGE_TO_SCAN = 'vote'   // Change to the service you want to scan, e.g. vote/result/worker/seed-data
        TAG = 'latest'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/ihrishikesh0896/example-voting-app.git'
            }
        }
        stage('Build Images') {
            steps {
                sh '''
                  docker compose build
                '''
            }
        }
        stage('Get Image ID for Scan') {
            steps {
                script {
                    def imageName = "${env.JOB_NAME}-${env.IMAGE_TO_SCAN}:${env.TAG}"
                    def IMAGE_ID = sh(
                        script: "docker images --no-trunc --format '{{.ID}}' ${imageName}",
                        returnStdout: true
                    ).trim()
                    env.IMAGE_ID = IMAGE_ID
                    echo "Will scan Docker image: ${imageName} (ID: ${env.IMAGE_ID})"
                }
            }
        }
        stage('Scan all images with Qualys') {
    steps {
        script {
            def services = ['vote', 'result', 'worker', 'seed-data']
            for (svc in services) {
                def imageName = "${env.JOB_NAME}-${svc}:${env.TAG}"
                def IMAGE_ID = sh(
                    script: "docker images --no-trunc --format '{{.ID}}' ${imageName}",
                    returnStdout: true
                ).trim()
                echo "Scanning image: ${imageName} (${IMAGE_ID})"
                getImageVulnsFromQualys(
                    useGlobalConfig: true,
                    imageIds: IMAGE_ID
                )
            }
        }
    }
}

    }
    post {
        always {
            sh 'docker images'
        }
    }
}
