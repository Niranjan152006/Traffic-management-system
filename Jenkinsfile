pipeline {
    agent any

    environment {
        DOCKER_HUB_USER = 'YOUR_DOCKERHUB_USERNAME'
        IMAGE_NAME      = 'traffic-dashboard'
        IMAGE_TAG       = "${BUILD_NUMBER}"
        KUBECONFIG_CRED = 'jenkins-kubeconfig' // Setup in Jenkins Credentials
    }

    parameters {
        choice(name: 'DEPLOY_VERSION', choices: ['STABLE', 'FAULTY_RELEASE'], description: 'Select deployment mode')
    }

    stages {
        stage('Pull Code & Static Analysis') {
            steps {
                echo 'Checking Repository Workspace Quality...'
                sh 'test -f index.html && test -f Dockerfile'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    if (params.DEPLOY_VERSION == 'FAULTY_RELEASE') {
                        echo 'Injecting faulty image structural tag setup...'
                        // Modifies the file with breaking syntax to simulate an app crash/error
                        sh "sed -i 's/window.onload = renderTable;/window.onload = function() { throw new Error(\"FAULTY CRASH\"); };/g' index.html"
                    }
                    sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ."
                    sh "docker tag ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Automated App Verification (Local Test)') {
            steps {
                echo 'Verifying container local startup status...'
                sh "docker run -d --name test_run -p 8085:80 ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                sh "sleep 5"
                // Basic curl test validation checking for running platform
                sh "curl -s http://localhost:8085 | grep -q 'Traffic' || (docker rm -f test_run && exit 1)"
                sh "docker rm -f test_run"
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                    sh "echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin"
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Deploy to Kubernetes Cluster') {
            steps {
                configFileProvider([configFile(fileId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG')]) {
                    script {
                        // Dynamically update deployment configuration matching image tag
                        sh "sed -i 's|${DOCKER_HUB_USER}/${IMAGE_NAME}:latest|${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}|g' deployment.yaml"
                        sh "kubectl apply -f deployment.yaml --kubeconfig=${KUBECONFIG}"
                    }
                }
            }
        }

        stage('Rollout Verification & Automated Rollback') {
            steps {
                configFileProvider([configFile(fileId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG')]) {
                    script {
                        echo "Monitoring Kubernetes rollout status..."
                        try {
                            // Check if Kubernetes successfully deploys pods within 60 seconds
                            sh "kubectl rollout status deployment/traffic-system-deployment --timeout=60s --kubeconfig=${KUBECONFIG}"
                            echo "Application Rollout Succeeded!"
                        } catch (Exception e) {
                            echo "⚠️ Deployment verification failed! Initiating automated recovery rollback..."
                            sh "kubectl rollout undo deployment/traffic-system-deployment --kubeconfig=${KUBECONFIG}"
                            sh "kubectl rollout status deployment/traffic-system-deployment --kubeconfig=${KUBECONFIG}"
                            error("Rollout aborted due to fault. Successfully rolled back to last stable version.")
                        }
                    }
                }
            }
        }
    }
}

