pipeline {
    agent any
    environment {
        IMAGE_NAME = 'sentiment-ai'
        REGISTRY = 'ghcr.io/henoc-kague'
        IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log --oneline -5'
            }
        }
        stage('Lint') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:lint ."
                sh "docker run --rm ${IMAGE_NAME}:lint sh -c 'pip install flake8 -q && flake8 src/ --max-line-length=100'"
            }
        }

        stage('IaC Validate') {
            steps {
                dir('infra') {
                    sh 'terraform init -backend=false -input=false'
                    sh 'terraform validate'
                }
            }
        }
        stage('Build & Test') {
            steps {
                sh '''
                    docker build -t $IMAGE_NAME:$IMAGE_TAG .
                    docker rm -f test-runner 2>/dev/null || true
                    set +e
                    docker run -e CI=true --name test-runner $IMAGE_NAME:$IMAGE_TAG pytest tests/ -v --cov=src --cov-report=xml:/tmp/coverage.xml --cov-report=term-missing --cov-fail-under=70
                    TEST_EXIT_CODE=$?
                    set -e
                    docker cp test-runner:/tmp/coverage.xml ./coverage.xml 2>/dev/null || true
                    docker rm -f test-runner 2>/dev/null || true
                    exit $TEST_EXIT_CODE
                '''
            }
            post {
                failure { echo 'Tests echoues ou coverage insuffisant' }
            }
        }
        stage('SonarQube Analysis') {
            environment {
                SONARQUBE_TOKEN = credentials('sonar-token')
            }
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        docker run --rm \
                            --network sentiment-ai_cicd-network \
                            -v $WORKSPACE:/app \
                            -w /app \
                            -e SONAR_HOST_URL=$SONAR_HOST_URL \
                            -e SONAR_TOKEN=$SONARQUBE_TOKEN \
                            sonarsource/sonar-scanner-cli:latest \
                            sonar-scanner \
                            -Dsonar.projectKey=sentiment-ai \
                            -Dsonar.projectName=SentimentAI \
                            -Dsonar.sources=. \
                            -Dsonar.inclusions=src/** \
                            -Dsonar.python.version=3.11 \
                            -Dsonar.python.coverage.reportPaths=coverage.xml \
                            -Dsonar.sourceEncoding=UTF-8
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    sleep(10)
                    def response = sh(script: "curl -s -u \${SONARQUBE_TOKEN}: http://sonarqube:9000/api/qualitygates/project_status?projectKey=sentiment-ai", returnStdout: true).trim()
                    echo "Quality Gate response: \${response}"
                    if (response.contains('"status":"ERROR"')) {
                        error('Quality Gate FAILED')
                    }
                    echo 'Quality Gate PASSED'
                }
            }
        }
        stage('Security Scan') {
            steps {
                sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache/trivy aquasec/trivy:latest image --severity HIGH,CRITICAL --exit-code 0 --format table $IMAGE_NAME:$IMAGE_TAG"
            }
            post {
                failure { echo 'Vulnerabilites CRITICAL ou HIGH detectees !' }
            }
        }
        stage('Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'REGISTRY_USER', passwordVariable: 'REGISTRY_PASS')]) {
                    sh "echo $REGISTRY_PASS | docker login ghcr.io -u $REGISTRY_USER --password-stdin"
                    sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:latest"
                    sh "docker push ${REGISTRY}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('IaC Apply') {
            steps {
                sh 'docker stop sentiment-staging 2>/dev/null || true'
                sh 'docker rm sentiment-staging 2>/dev/null || true'
                dir('infra') {
                    sh 'terraform init -input=false'
                    sh 'terraform state rm docker_container.sentiment_staging 2>/dev/null || true'
                    sh 'terraform state rm docker_image.sentiment 2>/dev/null || true'
                    sh "terraform apply -auto-approve -var=image_tag=${IMAGE_TAG}"
                }
            }
        }
        stage('Deploy Staging') {
            steps {
                sh 'echo "Staging deploye via Terraform sur http://localhost:8082"'
                sh 'docker ps | grep sentiment-staging'
            }
        }
    }
    post {
        always { sh 'docker compose down -v 2>/dev/null || true' }
        success { echo 'Pipeline reussi !' }
        failure { echo 'Pipeline echoue.' }
    }
}
