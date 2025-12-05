pipeline {
    agent any

    environment {
        SONAR_HOST_URL = "http://v2code.rtwohealthcare.com"
        SONAR_TOKEN    = "sqp_ab4016bc5eef902acdbc5f5dbf8f0d46815f0035"
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
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
                echo "Running Legacy Sonar Scanner (4.6) for old SonarQube"

                sh """
                    docker build -t sonar-scanner-custom -f sonar-runner-Dockerfile .
                    
                    docker run --rm \
                        -v ${WORKSPACE}:/src \
                        -w /src sonar-scanner-custom \
                        sonar-scanner \
                            -Dsonar.projectKey=test_v3 \
                            -Dsonar.sources=backend \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.login=${SONAR_TOKEN}
                """
            }
        }
    }
}
