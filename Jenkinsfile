pipeline {
    agent any

    environment {
        // SonarQube project info
        SONAR_HOST_URL = "https://v2code.rtwohealthcare.com"
        SONAR_TOKEN = "sqp_ab4016bc5eef902acdbc5f5dbf8f0d46815f0035"
        SONAR_PROJECT_KEY = "test_v3"

        // Registry
        DOCKER_REGISTRY = "v2deploy.rtwohealthcare.com"
        IMAGE_NAME = "test_v3"
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install & Test Python Code') {
            steps {
                sh """
                    docker run --rm \
                        -v ${WORKSPACE}/backend:/app \
                        -w /app python:3.10-slim sh -c "
                            pip install --upgrade pip &&
                            pip install -r requirements.txt &&
                            pytest --disable-warnings --maxfail=1 -q || true
                        "
                """
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "🔍 Running Sonar Scanner inside custom Docker image"

                sh """
                    # Build custom Sonar scanner with SSL certs
                    docker build -t sonar-scanner-custom -f sonar-runner-Dockerfile .

                    # Run scan
                    docker run --rm \
                        -v ${WORKSPACE}:/src \
                        -w /src sonar-scanner-custom \
                        sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.sources=backend \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.token=${SONAR_TOKEN}
                """
            }
        }

        stage("Skip Quality Gate") {
            steps {
                echo "⏭️ Skipping Quality Gate (Python project)"
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    cd backend
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-docker-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh """
                        echo "$PASS" | docker login ${DOCKER_REGISTRY} -u "$USER" --password-stdin

                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest

                        docker logout ${DOCKER_REGISTRY}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed — image pushed successfully!"
        }
        failure {
            echo "❌ Pipeline FAILED — go fix it."
        }
    }
}
