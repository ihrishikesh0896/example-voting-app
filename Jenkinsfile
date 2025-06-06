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
                sh 'docker compose build'
            }
        }
        stage('Parallel Image Scan') {
            steps {
                script {
                    def services = ['vote', 'result', 'worker', 'seed-data']
                    def scanStages = [:]

                    for (svc in services) {
                        // IMPORTANT: closure variable must be final/effectively final for parallel
                        def svcName = svc
                        scanStages["Scan ${svcName}"] = {
                            def imageName = "${env.JOB_NAME}-${svcName}:${env.TAG}"
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
        stage('Generate SBOMs (Syft)') {
            steps {
                script {
                    def services = ['vote', 'result', 'worker', 'seed-data']
                    for (svc in services) {
                        def imageName = "${env.JOB_NAME}-${svc}:${env.TAG}"
                        def sbomFile = "sbom-${svc}.json"
                        echo "📝 Generating SBOM for ${imageName}"
                        sh "syft packages docker:${imageName} -o cyclonedx-json > ${sbomFile} || echo 'Syft failed for ${imageName}'"
                        archiveArtifacts artifacts: "${sbomFile}", allowEmptyArchive: true
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
