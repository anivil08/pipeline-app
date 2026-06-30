pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY_USER = 'anivil08'
        REGISTRY_CREDENTIALS_ID = 'docker-hub-credentials'
        IMAGE_NAME = 'pipeline-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Pull Code') {
            steps {
                echo '📥 Synchronizing Source Repositories...'
                checkout scm
            }
        }
        
        stage('Build Image') {
            steps {
                echo '🛠️ Executing Docker Engine Container Build...'
                sh "docker build -t ${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "docker build -t ${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:latest ."
            }
        }
        
        stage('Publish Image') {
            steps {
                echo '🚀 Uploading Image to Central Registry...'
                withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS_ID}", passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USERNAME')]) {
                    sh "echo \$DOCKER_HUB_PASSWORD | docker login -u \$DOCKER_HUB_USERNAME --password-stdin"
                    sh "docker push ${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:latest"
                }
            }
        }
        
        stage('Remote Cluster Deployment') {
            steps {
                echo '🚢 Injecting Blueprints into Remote Kubespray Nodes...'
                sh "sed -i 's|anivil08/pipeline-app:IMAGE_TAG|${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:${IMAGE_TAG}|g' k8s-deployment.yaml"
                
                withCredentials([file(credentialsId: 'k8s-kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh "kubectl --kubeconfig=\$KUBECONFIG_FILE apply -f k8s-deployment.yaml"
                }
            }
        }
        
        stage('Verify Zero-Downtime Rollout') {
            steps {
                echo '🔍 Validating Container Routing State...'
                withCredentials([file(credentialsId: 'k8s-kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh "kubectl --kubeconfig=\$KUBECONFIG_FILE rollout status deployment/pipeline-web-app --timeout=60s"
                }
            }
        }
    }
}
