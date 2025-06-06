pipeline {
    agent any

    environment {
        DOCKER_COMPOSE_VERSION = '2.24.6'
        REGISTRY = 'your-registry.com'      // Change to your actual registry
        TAG = 'latest'
        SERVICES = 'vote,result,worker'     // Comma-separated, no spaces
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                echo "🔄 Checking out source code..."
                checkout scm
            }
        }

        stage('Pre-build Cleanup') {
            steps {
                echo "🧹 Cleaning up previous builds and images..."
                sh '''
                    docker system prune -f --volumes || true
                    docker compose down --remove-orphans || true
                '''
            }
        }

        stage('Build All Services') {
            steps {
                echo "🏗️ Building all Docker services with latest tag..."
                sh '''
                    docker compose build --no-cache --parallel
                    echo "=== Built Images ==="
                    docker images
                '''
            }
        }

        stage('Validate Built Images') {
            steps {
                script {
                    echo "✅ Validating built images with latest tag..."
                    def services = env.SERVICES.split(',')
                    def missingImages = []
                    def foundImages = []

                    // Print all images for troubleshooting
                    sh 'docker images'

                    services.each { svc ->
                        def service = svc.trim()
                        def imageName = "${env.JOB_NAME}-${service}:${env.TAG}"

                        // Robust check using returnStatus
                        def status = sh(
                            script: "docker images --format '{{.Repository}}:{{.Tag}}' | grep -w '${imageName}'",
                            returnStatus: true
                        )
                        if (status == 0) {
                            echo "✓ Found image: ${imageName}"
                            foundImages << imageName
                        } else {
                            echo "❌ Missing image: ${imageName}"
                            missingImages << imageName
                        }
                    }

                    if (missingImages) {
                        error("❌ Missing images for: ${missingImages.join(', ')}")
                    }
                    echo "✅ All required images validated: ${foundImages.join(', ')}"
                }
            }
        }

        stage('Security Scanning') {
            parallel {
                stage('Scan Vote Service') {
                    steps {
                        script {
                            scanImageWithQualys('vote')
                        }
                    }
                }
                stage('Scan Result Service') {
                    steps {
                        script {
                            scanImageWithQualys('result')
                        }
                    }
                }
                stage('Scan Worker Service') {
                    steps {
                        script {
                            scanImageWithQualys('worker')
                        }
                    }
                }
            }
        }

        stage('Generate Security Report') {
            steps {
                echo "📊 (Optional) Generate consolidated security report here."
                // Archive or publish scan results as needed by your Qualys plugin/output
                // archiveArtifacts artifacts: '**/qualys-scan-*.json', allowEmptyArchive: true
            }
        }

        stage('Tag and Push Images') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    echo "🏷️ Tagging and pushing latest images..."
                    def services = env.SERVICES.split(',')

                    services.each { svc ->
                        def service = svc.trim()
                        def localImage = "${env.JOB_NAME}-${service}:${env.TAG}"
                        def remoteImage = "${env.REGISTRY}/${service}:${env.TAG}"
                        def timestampImage = "${env.REGISTRY}/${service}:${env.BUILD_NUMBER}"

                        sh """
                            docker tag ${localImage} ${remoteImage}
                            docker push ${remoteImage}
                            echo "✓ Pushed ${remoteImage}"

                            docker tag ${localImage} ${timestampImage}
                            docker push ${timestampImage}
                            echo "✓ Pushed ${timestampImage}"
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo "📋 Pipeline execution summary:"
            sh '''
                echo "=== Built Images ==="
                docker images | grep ${JOB_NAME} || echo "No images found"
                echo "=== Docker System Info ==="
                docker system df
            '''
        }
        success {
            echo "✅ Pipeline completed successfully!"
            // Add notifications here if needed
        }
        failure {
            echo "❌ Pipeline failed! Check the log above for details."
            // Add notifications here if needed
        }
        cleanup {
            echo "🧹 Cleaning up resources..."
            sh '''
                docker compose down --remove-orphans || true
                docker image prune -f || true
                find . -name "*.log" -mtime +3 -delete || true
            '''
        }
    }
}

// Helper function for scanning with Qualys
def scanImageWithQualys(String serviceName) {
    def imageName = "${env.JOB_NAME}-${serviceName}:${env.TAG}"
    def IMAGE_ID = sh(
        script: "docker images --no-trunc --format '{{.ID}}' ${imageName}",
        returnStdout: true
    ).trim()

    if (IMAGE_ID) {
        echo "🔍 Scanning ${serviceName} service (Image: ${imageName}, ID: ${IMAGE_ID})"
        // Adjust parameters below as per your Qualys plugin version
        getImageVulnsFromQualys(
            useGlobalConfig: true,
            imageIds: IMAGE_ID,
            isSev3Vulns: true, // Medium
            isSev4Vulns: true, // High
            isSev5Vulns: true  // Critical
        )
        echo "✅ Scan completed for ${serviceName}"
    } else {
        error("❌ Could not find image ID for ${imageName}")
    }
}
