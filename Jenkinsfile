
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = 'flask-registration'
        COMPOSE_FILE = 'docker-compose.yml'
    }

    stages {

        stage('1. Checkout') {
            steps {
                checkout scm
            }
        }

        stage('2. Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv .venv
                    .venv/bin/pip install --upgrade pip
                    .venv/bin/pip install -r requirements.txt
                '''
            }
        }

        stage('3. Unit Tests') {
            steps {
                sh '''
                    .venv/bin/pytest -v
                '''
            }
        }

        stage('4. Security Scan') {
            steps {
                sh '''
                    .venv/bin/pip install bandit pip-audit

                    echo "Running Bandit Security Scan..."

                    bandit -r app.py -lll \
                        -f json -o bandit-report.json

                    echo "Running Dependency Audit..."

                    .venv/bin/pip-audit \
                        -r requirements.txt \
                        -f json -o pip-audit-report.json
                '''
            }
        }

        stage('5. Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .

                    docker tag ${IMAGE_NAME}:${BUILD_NUMBER} \
                        ${IMAGE_NAME}:latest

                    docker images | grep flask-registration
                '''
            }
        }

        stage('6. Deploy') {
            steps {
                sh '''
                    if docker compose version >/dev/null 2>&1; then
                        docker compose -f ${COMPOSE_FILE} up -d --build

                    elif command -v docker-compose >/dev/null 2>&1; then
                        docker-compose -f ${COMPOSE_FILE} up -d --build

                    else
                        echo "Docker Compose is not installed!"
                        exit 1
                    fi
                '''
            }
        }

        stage('7. Smoke Test') {
            steps {
                sh '''
                    echo "Checking Flask Application..."

                    for i in $(seq 1 30); do

                        if curl -fsS http://localhost:5000/health; then
                            echo "Application is Healthy!"
                            exit 0
                        fi

                        sleep 5
                    done

                    echo "Application health check failed"
                    exit 1
                '''
            }
        }
    }

    post {

        always {
            sh '''
                if docker compose version >/dev/null 2>&1; then
                    docker compose -f ${COMPOSE_FILE} ps || true

                elif command -v docker-compose >/dev/null 2>&1; then
                    docker-compose -f ${COMPOSE_FILE} ps || true
                fi
            '''

            archiveArtifacts artifacts: '*.json,pytest.ini',
                allowEmptyArchive: true
        }

        success {
            echo 'Flask Registration Application deployed successfully!'
        }

        failure {
            echo 'Pipeline failed. Review the stage logs.'
        }
    }
}
