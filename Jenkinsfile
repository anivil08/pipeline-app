pipeline {
    agent any

    environment {
        DOCKER_REGISTRY_USER = 'anivil08'
        REGISTRY_CREDENTIALS_ID = 'docker-hub-credentials'
        IMAGE_NAME = 'pipeline-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        SONAR_SCANNER_HOME = tool 'SonarQubeScanner'
    }

    stages {

        stage('Pull Code') {
            steps {
                echo '📥 Synchronizing Source Repositories...'
                checkout scm
            }
        }

        stage('Security: Secret Detection') {
            steps {
                sh '''
                    trivy fs \
                      --scanners secret \
                      --exit-code 1 \
                      .
                '''
            }
        }

        stage('Security: SonarQube SAST') {
            steps {
                withSonarQubeEnv('SonarQubeServer') {
                    sh '''
                        ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=pipeline-app \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=**/node_modules/**,dependency-check-report/**
                    '''
                }

                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Dependency Scan - OWASP') {
            steps {
                sh '''
                    echo "🔍 Running OWASP Dependency-Check..."

                    rm -rf dependency-check-report
                    mkdir -p dependency-check-report

                    dependency-check.sh \
                      --project "$IMAGE_NAME" \
                      --scan . \
                      --format HTML \
                      --format JSON \
                      --format XML \
                      --out dependency-check-report \
                      --data /opt/dependency-check/data \
                      --noupdate \
                      --failOnCVSS 7 \
                      --exclude "**/node_modules/**"

                    echo "✅ OWASP Dependency-Check completed"
                    echo "📄 Generated reports:"
                    ls -lh dependency-check-report/
                }
            }

            post {
                always {
                    dependencyCheckPublisher(
                        pattern: 'dependency-check-report/dependency-check-report.xml'
                    )

                    archiveArtifacts(
                        artifacts: 'dependency-check-report/*',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Build Image') {
            steps {
                echo '🛠️ Executing Docker Engine Container Build...'

                sh '''
                    docker build \
                      -t ${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:${IMAGE_TAG} \
                      .

                    docker build \
                      -t ${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:latest \
                      .
                '''
            }
        }

        stage('Security: Trivy Container Scan') {
            steps {
                echo '🔐 Scanning Docker image for HIGH and CRITICAL vulnerabilities...'

                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      ${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Publish Image') {
            steps {
                echo '🚀 Uploading Image to Central Registry...'

                withCredentials([
                    usernamePassword(
                        credentialsId: "${REGISTRY_CREDENTIALS_ID}",
                        passwordVariable: 'DOCKER_HUB_PASSWORD',
                        usernameVariable: 'DOCKER_HUB_USERNAME'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_HUB_PASSWORD" | \
                          docker login \
                          -u "$DOCKER_HUB_USERNAME" \
                          --password-stdin

                        docker push \
                          ${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:${IMAGE_TAG}

                        docker push \
                          ${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Remote Cluster Deployment') {
            steps {
                echo '🚢 Deploying application to Kubernetes cluster...'

                sh '''
                    sed -i \
                      "s|anivil08/pipeline-app:IMAGE_TAG|${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:${IMAGE_TAG}|g" \
                      k8s-deployment.yaml
                '''

                withCredentials([
                    file(
                        credentialsId: 'k8s-kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {

                    sh '''
                        kubectl \
                          --kubeconfig="$KUBECONFIG_FILE" \
                          apply \
                          -f k8s-deployment.yaml
                    '''
                }
            }
        }

        stage('Verify Zero-Downtime Rollout') {
            steps {
                echo '🔍 Validating Kubernetes rollout status...'

                withCredentials([
                    file(
                        credentialsId: 'k8s-kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {

                    sh '''
                        kubectl \
                          --kubeconfig="$KUBECONFIG_FILE" \
                          rollout status \
                          deployment/pipeline-web-app \
                          --timeout=60s
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ CI/CD Pipeline completed successfully!'
            echo "🚀 Image: ${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
        }

        failure {
            echo '❌ CI/CD Pipeline failed. Check the failed stage and logs above.'
        }

        always {
            echo "🏁 Pipeline completed with status: ${currentBuild.currentResult}"
        }
    }
}
