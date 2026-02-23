pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_IMAGE = "captainnoor1/devops-examapp:${BUILD_NUMBER}"
        SCANNER_HOME = tool 'sonar-scanner'
        COMPOSE_PROJECT_NAME = "devopsexamapp"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/KastroVKiran/devops-exam-app.git',
                    branch: 'master'
            }
        }

        stage('Verify Docker') {
            steps {
                sh 'docker --version'
                sh 'docker compose version'
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                trivy fs --scanners vuln,misconfig \
                --format table \
                -o trivy-report.txt .
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectName=devops-exam-app \
                    -Dsonar.projectKey=devops-exam-app \
                    -Dsonar.sources=. \
                    -Dsonar.python.version=3
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('backend') {
                    script {
                        withDockerRegistry(credentialsId: 'docker') {
                            sh "docker build -t ${DOCKER_IMAGE} ."
                        }
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                export COMPOSE_PROJECT_NAME=${COMPOSE_PROJECT_NAME}
                export IMAGE_TAG=${IMAGE_TAG}

                echo "Stopping old deployment..."
                docker compose down --remove-orphans --volumes || true

                echo "Pulling latest image..."
                docker compose pull

                echo "Starting new containers..."
                docker compose up -d --force-recreate

                echo "Cleaning old images..."
                docker image prune -f
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                docker compose ps
                curl -I http://localhost:5000 || true
                '''
            }
        }
    }

    post {
        success {
            echo "🚀 Deployment successful!"
        }
        failure {
            echo "❗ Deployment failed. Logs:"
            sh 'docker compose logs --tail=50 || true'
        }
    }
}
