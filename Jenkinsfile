pipeline {
    agent any

    environment {
        DOCKER_COMPOSE_VERSION = '2.24.6'
        REGISTRY = 'your-registry.com'  // Optional: for pushing images
        TAG = 'latest'  // Always use latest
        SERVICES = 'vote,result,worker'  // Removed seed-data since it's not building
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "🔄 Checking out source code..."
                    checkout scm
                }
            }
        }

        stage('Pre-build Cleanup') {
            steps {
                script {
                    echo "🧹 Cleaning up previous builds..."
                    sh '''
                        docker system prune -f --volumes || true
                        docker compose down --remove-orphans || true
                    '''
                }
            }
        }

        stage('Build All Services') {
            steps {
                script {
                    echo "🏗️ Building all Docker services with latest tag..."
                    sh '''
                        docker compose build --no-cache --parallel
                        
                        echo "=== Built Images ==="
                        docker images | grep ${JOB_NAME} || echo "No images found with job name"
                    '''
                }
            }
        }

        stage('Validate Built Images') {
            steps {
                script {
                    echo "✅ Validating built images with latest tag..."
                    def services = env.SERVICES.split(',')
                    def missingImages = []

                    services.each { service ->
                        def imageName = "${env.JOB_NAME}-${service.trim()}:latest"
                        def imageExists = sh(
                            script: "docker images --format '{{.Repository}}:{{.Tag}}' | grep -q '${imageName}' && echo 'true' || echo 'false'",
                            returnStdout: true
                        ).trim()

                        if (imageExists == 'false') {
                            missingImages.add(service.trim())
                        } else {
                            echo "✓ Found image: ${imageName}"
                        }
                    }

                    if (missingImages.size() > 0) {
                        error("❌ Missing images for services: ${missingImages.join(', ')}")
                    } else {
                        echo "✅ All required images validated successfully!"
                    }
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
                script {
                    echo "📊 Generating consolidated security report..."
                    // Archive scan results if available
                    archiveArtifacts artifacts: '**/qualys-scan-*.json', allowEmptyArchive: true

                    // Publish security report (if using plugins like HTML Publisher)
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports',
                        reportFiles: 'security-report.html',
                        reportName: 'Security Scan Report'
                    ])
                }
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

                    services.each { service ->
                        def serviceName = service.trim()
                        def localImage = "${env.JOB_NAME}-${serviceName}:latest"
                        def remoteImage = "${env.REGISTRY}/${serviceName}:latest"
                        def timestampImage = "${env.REGISTRY}/${serviceName}:${env.BUILD_NUMBER}"

                        sh """
                            # Push with latest tag
                            docker tag ${localImage} ${remoteImage}
                            docker push ${remoteImage}
                            echo "✓ Pushed ${remoteImage}"
                            
                            # Also push with build number for versioning
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
            script {
                echo "📋 Pipeline execution summary:"
                sh '''
                    echo "=== Built Images ==="
                    docker images | grep ${JOB_NAME} || echo "No images found"

                    echo "=== Docker System Info ==="
                    docker system df
                '''
            }
        }

        success {
            script {
                echo "✅ Pipeline completed successfully!"
                // Send success notification
                emailext (
                    subject: "✅ Security Scan Passed: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                    body: """
                        Pipeline: ${env.JOB_NAME}
                        Build: #${env.BUILD_NUMBER}
                        Status: SUCCESS

                        All services have been built and scanned successfully.
                        Scanned services: ${env.SERVICES}

                        View full report: ${env.BUILD_URL}
                    """,
                    to: "${env.CHANGE_AUTHOR_EMAIL ?: 'team@company.com'}"
                )
            }
        }

        failure {
            script {
                echo "❌ Pipeline failed!"
                // Send failure notification
                emailext (
                    subject: "❌ Security Scan Failed: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                    body: """
                        Pipeline: ${env.JOB_NAME}
                        Build: #${env.BUILD_NUMBER}
                        Status: FAILED

                        Please check the build logs for details.

                        View logs: ${env.BUILD_URL}console
                    """,
                    to: "${env.CHANGE_AUTHOR_EMAIL ?: 'team@company.com'}"
                )
            }
        }

        cleanup {
            script {
                echo "🧹 Cleaning up resources..."
                sh '''
                    # Clean up containers and networks
                    docker compose down --remove-orphans || true

                    # Remove dangling images
                    docker image prune -f || true

                    # Clean up build artifacts (keep last 3 builds)
                    find . -name "*.log" -mtime +3 -delete || true
                '''
            }
        }
    }
}

// Helper function to scan images with Qualys - Updated for latest tag
def scanImageWithQualys(String serviceName) {
    try {
        def imageName = "${env.JOB_NAME}-${serviceName}:latest"
        def IMAGE_ID = sh(
            script: "docker images --no-trunc --format '{{.ID}}' ${imageName}",
            returnStdout: true
        ).trim()

        if (IMAGE_ID) {
            echo "🔍 Scanning ${serviceName} service (Image: ${imageName}, ID: ${IMAGE_ID})"

            // Perform Qualys scan
            getImageVulnsFromQualys(
                useGlobalConfig: true,
                imageIds: IMAGE_ID,
                severity: 'Medium,High,Critical'  // Only scan for medium and above
            )

            echo "✅ Scan completed for ${serviceName}"
        } else {
            error("❌ Could not find image ID for ${imageName}")
        }

    } catch (Exception e) {
        echo "❌ Scan failed for ${serviceName}: ${e.getMessage()}"
        currentBuild.result = 'UNSTABLE'
        throw e
    }
}
