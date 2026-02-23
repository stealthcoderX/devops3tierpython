pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "captainnoor1/devops-examapp:latest"
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/KastroVKiran/devops-exam-app.git',
                    branch: 'master'
            }
        }

        stage('Verify Docker Compose') {
            steps {
                sh '''
                docker compose version || { echo "Docker Compose not available"; exit 1; }
                '''
            }
        }

        stage('File System Scan (Trivy)') {
            steps {
                sh '''
                trivy fs --security-checks vuln,config \
                --format table \
                -o trivy-fs-report.txt .
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

        stage('Push to Docker Hub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Docker Scout Analysis') {
            steps {
                sh """
                docker scout quickview ${DOCKER_IMAGE} || true
                docker scout cves ${DOCKER_IMAGE} || true
                docker scout recommendations ${DOCKER_IMAGE} || true
                """
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                sh '''
                docker compose down --remove-orphans || true
                docker compose up -d

                echo "Waiting for MySQL..."
                timeout 120s bash -c '
                until docker compose exec -T mysql mysqladmin ping -uroot -prootpass --silent;
                do
                    sleep 5
                done'

                sleep 10
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                echo "=== Container Status ==="
                docker compose ps -a

                echo "=== Testing Flask Endpoint ==="
                curl -I http://localhost:5000 || true
                '''
            }
        }
    }

    post {
        success {
            echo '🚀 Deployment successful!'
            sh 'docker compose ps'
            sh 'docker images | grep devops-examapp || true'
        }
        failure {
            echo '❗ Pipeline failed. Check logs above.'
            sh 'docker compose logs --tail=50 || true'
        }
        always {
            sh 'docker compose logs --tail=20 || true'
        }
    }
}
