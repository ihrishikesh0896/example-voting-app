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
        stage('Parallel Image Scan') {
            steps {
                script {
                    def services = ['vote', 'result', 'worker', 'seed-data']
                    def scanStages = [:]

                    for (svc in services) {
                        def imageName = "${env.JOB_NAME}-${svc}:${env.TAG}"
                        // Important: use a variable inside the closure so each gets its own value
                        scanStages["Scan ${svc}"] = {
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

                    parallel scanStages
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
