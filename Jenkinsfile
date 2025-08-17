pipeline {
    agent any
    tools {
        nodejs 'nodejs'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        FRONTEND_DIR = 'client'
        BACKEND_DIR = 'api'
        DOCKER_REGISTRY = 'docker.io/leoworths'
        IMAGE_TAG = "v1.0.${env.BUILD_NUMBER}"
        PROJECT_NAME = '3-Tier-Devsecops-Project'
        ENVIRONMENT = 'Production'
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/leoworths/3-Tier-DevSecOps-Project.git'
            }
        }
        stage('Code Syntax Check') {
            parallel {
                stage('Frontend Syntax Check') {
                    steps {
                        dir("${FRONTEND_DIR}") {
                            sh 'find . -name "*.js" -exec node --check {} +'
                        }
                    }
                }
                stage('Backend Syntax Check') {
                    steps {
                        dir("${BACKEND_DIR}") {
                            sh 'find . -name "*.js" -exec node --check {} +'
                        }
                    }
                }
            }
        }
                stage('Install & Test Dependencies '){
                    parallel {
                        stage('Frontend Dependencies') {
                            steps {
                                dir("${FRONTEND_DIR}") {
                                    sh 'npm install'
                                    sh 'npm run build'
                                    sh 'npm test'
                                }
                            }
                        }
                        stage('Backend Dependencies') {
                            steps {
                                dir("${BACKEND_DIR}") {
                                    sh 'npm install'
                                    sh 'npm run build'
                                    sh 'npm test'
                                }
                            }
                        }
                    }
                }
                stage('Security Check & Vulnerability Scan') {
                    parallel {
                        stage('GitLeaks Scan') {
                            steps {
                                sh 'gitleaks detect --source ./client --exit-code 1'
                                sh 'gitleaks detect --source ./api --exit-code 1'
                            }
                        }
                        stage('Trivy FS Scan') {
                            steps {
                                sh 'trivy fs --format table -o fs-report.html .'
                            }
                        }
                    }
                }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """ $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=NodeJS-Project \
                            -Dsonar.projectKey=NodeJS-Project """
                }
            }
        }
        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Build-Tag Scan & Push Docker Images') {
            parallel {
                stage('Frontend Docker Build') {
                    steps {
                        dir("${FRONTEND_DIR}") {
                            script {
                                def frontendImage = "${DOCKER_REGISTRY}/frontend:${IMAGE_TAG}"
                                withDockerRegistry(credentialsId: 'docker-cred' ) {
                                    sh "docker build -t ${frontendImage} ."
                                    sh "trivy image --format table -o frontend-report.html ${frontendImage}"
                                    sh "docker push ${frontendImage}"
                                }
                            }
                        }
                    }
                }
                stage('Backend Docker Build') {
                    steps {
                        dir("${BACKEND_DIR}") {
                            script {
                                def backendImage = "${DOCKER_REGISTRY}/backend:${IMAGE_TAG}"
                                withDockerRegistry(credentialsId: 'docker-cred' ) {
                                    sh "docker build -t ${backendImage} ."
                                    sh "trivy image --format table -o backend-report.html ${backendImage}"
                                    sh "docker push ${backendImage}"                                    
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Update Kubernetes In Git') {
            steps {
                script {
                    def backendImage = "${DOCKER_REGISTRY}/backend:${IMAGE_TAG}"
                    def frontendImage = "${DOCKER_REGISTRY}/frontend:${IMAGE_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'git-cred', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh "git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/leoworths/3-Tier-Devsecops-Project.git"
                        dir('3-Tier-Devsecops-Project') {
                            sh "sed -i 's|image: .*|image: ${backendImage}|' k8s-prod/backend.yaml"
                            sh "sed -i 's|image: .*|image: ${frontendImage}|' k8s-prod/frontend.yaml"
                            sh "git config --global user.email 'leoworth@gmail.com'"
                            sh "git config --global user.name 'leoworths'"
                            sh "git add k8s-prod/backend.yaml k8s-prod/frontend.yaml"
                            sh "git commit -m 'Update backend and frontend images'" 
                            sh "git push origin main"
                        }
                    }
                }
            }
        }
        stage('Cleanup Docker Images') {
            steps {
                script {
                    def backendImage = "${DOCKER_REGISTRY}/backend:${IMAGE_TAG}"
                    def frontendImage = "${DOCKER_REGISTRY}/frontend:${IMAGE_TAG}"
                    sh "docker rmi -f ${backendImage}"
                    sh "docker rmi -f ${frontendImage}"
                    // Remove dangling images
                    def danglingImages = sh(script: "docker images -q --filter dangling=true", returnStdout: true).trim()
                    if (danglingImages) {
                        sh "docker rmi -f ${danglingImages}"
                    }
                }
            }
        }
    }
    post {
        success {
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
                script {
                    def message = """{
                        "text": "*✅ ${PROJECT_NAME} Build Successful!*",
                        "attachments": [
                            {
                                "color": "#36a64f",
                                "fields": [
                                    { "title": "Job", "value": "${env.JOB_NAME}", "short": true },
                                    { "title": "Build", "value": "#${env.BUILD_NUMBER}", "short": true },
                                    { "title": "Environment", "value": "${ENVIRONMENT}", "short": true }
                                ],
                                "footer": "Jenkins CI",
                                "footer_icon": "https://www.jenkins.io/images/logos/jenkins/jenkins.png",
                                "ts": ${System.currentTimeMillis() / 1000},
                                "actions": [
                                    {
                                        "type": "button",
                                        "text": "View Build",
                                        "url": "${env.BUILD_URL}",
                                        "style": "primary"
                                    }
                                ]
                            }
                        ]
                    }"""
                    sh """curl -X POST -H 'Content-type: application/json' --data '${message}' $SLACK_URL"""
                }
            }
        }
        failure {
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
                script {
                    def message = """{
                        "text": "<!here> *❌ ${PROJECT_NAME} Build Failed!*",
                        "attachments": [
                            {
                                "color": "#FF0000",
                                "fields": [
                                    { "title": "Job", "value": "${env.JOB_NAME}", "short": true },
                                    { "title": "Build", "value": "#${env.BUILD_NUMBER}", "short": true },
                                    { "title": "Environment", "value": "${ENVIRONMENT}", "short": true }
                                ],
                                "footer": "Jenkins CI",
                                "footer_icon": "https://www.jenkins.io/images/logos/jenkins/jenkins.png",
                                "ts": ${System.currentTimeMillis() / 1000},
                                "actions": [
                                    {
                                        "type": "button",
                                        "text": "View Build Logs",
                                        "url": "${env.BUILD_URL}",
                                        "style": "danger"
                                    }
                                ]
                            }
                        ]
                    }"""
                    sh """curl -X POST -H 'Content-type: application/json' --data '${message}' $SLACK_URL"""
                }
            }
        }
        always {
            cleanWs()
        }
    }
}