pipeline {
    agent any
    environment {
        DOCKER_USER = 'your-dockerhub-username' // Change to your username
        IMAGE_NAME  = 'traffic-dashboard'
    }
    parameters {
        choice(name: 'BUILD_TYPE', choices: ['STABLE', 'FAULTY'], description: 'Deploy Stable or Faulty release?')
    }
    stages {
        stage('Initial Check') {
            steps {
                sh 'test -f index.html && test -f Dockerfile'
                echo 'Required files verified.'
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    if (params.BUILD_TYPE == 'FAULTY') {
                        echo 'Injecting faulty broken script...'
                        sh "sed -i 's/window.onload = renderTable;/window.onload = function(){throw new Error(\"FAIL\");};/g' index.html"
                    }
                    sh "docker build -t ${DOCKER_USER}/${IMAGE_NAME}:latest ."
                }
            }
        }
        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    sh "echo ${PASS} | docker login -u ${USER} --password-stdin"
                    sh "docker push ${DOCKER_USER}/${IMAGE_NAME}:latest"
                }
            }
        }
        stage('Kubernetes Deploy') {
            steps {
                sh "kubectl apply -f deployment.yaml"
            }
        }
        stage('Verify & Rollback') {
            steps {
                script {
                    try {
                        echo 'Verifying rollout state...'
                        sh 'kubectl rollout status deployment/traffic-deployment --timeout=20s'
                        
                        if (params.BUILD_TYPE == 'FAULTY') {
                            error "Simulated Application failure detected!"
                        }
                    } catch (Exception e) {
                        echo '🛑 Error detected! Automatically rolling back to the previous stable release...'
                        sh 'kubectl rollout undo deployment/traffic-deployment'
                        sh 'kubectl rollout status deployment/traffic-deployment'
                    }
                }
            }
        }
    }
}
