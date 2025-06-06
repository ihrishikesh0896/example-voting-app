pipeline {
    agent any

    environment {
        DOCKER_COMPOSE_VERSION = '2.24.6'
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
                        if (IMAGE_ID) {
                            echo "🔍 Scanning image: ${imageName} (${IMAGE_ID})"
                            getImageVulnsFromQualys(
                                useGlobalConfig: true,
                                imageIds: IMAGE_ID
                            )
                        } else {
                            echo "⏭️  Skipping scan: ${imageName} not built."
                        }
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
