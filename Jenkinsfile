pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "captainnoor1/devops-examapp:latest"
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

        stage('Verify Docker & Compose') {
            steps {
                sh '''
                docker --version
                docker compose version
                '''
            }
        }

        stage('File System Scan (Trivy)') {
            steps {
                sh '''
                trivy fs --scanners vuln,misconfig \
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

        stage('Docker Scout Analysis (Optional)') {
            steps {
                sh '''
                docker scout version >/dev/null 2>&1 || exit 0

                docker scout quickview ${DOCKER_IMAGE} || true
                docker scout cves ${DOCKER_IMAGE} || true
                docker scout recommendations ${DOCKER_IMAGE} || true
                '''
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                sh '''
                export COMPOSE_PROJECT_NAME=${COMPOSE_PROJECT_NAME}

                echo "Cleaning previous deployment..."
                docker compose down --remove-orphans --volumes || true

                echo "Force removing conflicting containers..."
                docker rm -f flask_app mysql_db 2>/dev/null || true

                echo "Starting fresh deployment..."
                docker compose up -d --build

                echo "Waiting for MySQL to be ready..."
                timeout 120s bash -c '
                until docker compose exec -T mysql mysqladmin ping -uroot -prootpass --silent;
                do
                    sleep 5
                done'

                echo "Deployment completed."
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                echo "=== Container Status ==="
                docker compose ps

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
        }
        failure {
            echo '❗ Pipeline failed. Investigating logs...'
            sh 'docker compose logs --tail=50 || true'
        }
        always {
            sh 'docker compose logs --tail=20 || true'
        }
    }
}
