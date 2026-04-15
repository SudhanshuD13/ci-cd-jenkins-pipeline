pipeline {
    agent any

    environment {
        DOCKER_IMAGE  = "sudhanshud100/flask-app"
        DOCKER_TAG    = "${BUILD_NUMBER}"
        SONAR_PROJECT = "flask-app"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ Code checked out"
            }
        }

        stage('Gitleaks - Secret Scan') {
            steps {
                sh '''
                    docker run --rm \
                        --user root \
                        -e GIT_DISCOVERY_ACROSS_FILESYSTEM=1 \
                        -v $(pwd):/path \
                        zricethezav/gitleaks:latest \
                        sh -c "git config --global --add safe.directory /path && gitleaks detect --source /path --report-format json --report-path /path/gitleaks-report.json --exit-code 1" || true

                    [ -f gitleaks-report.json ] || touch gitleaks-report.json
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('SonarQube - Code Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    withCredentials([
                        string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')
                    ]) {
                        sh '''
                            docker run --rm \
                                --user root \
                                --network ci-cd-jenkins-pipeline_devops-net \
                                -v $(pwd):/usr/src \
                                sonarsource/sonar-scanner-cli \
                                -Dsonar.projectKey=${SONAR_PROJECT} \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=http://sonarqube:9000 \
                                -Dsonar.token=$SONAR_TOKEN \
                                -Dsonar.working.directory=/usr/src/.scannerwork

                            if [ -f .scannerwork/report-task.txt ]; then
                                mv .scannerwork/report-task.txt .
                                echo "✅ report-task.txt moved to root"
                            fi
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                        -t ${DOCKER_IMAGE}:${DOCKER_TAG} \
                        -t ${DOCKER_IMAGE}:latest \
                        ./app
                '''
                echo "✅ Image built: ${DOCKER_IMAGE}:${DOCKER_TAG}"
            }
        }

        stage('Trivy - Image Scan') {
            steps {
                sh '''
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v $(pwd):/output \
                        aquasec/trivy:latest image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format json \
                        --output /output/trivy-report.json \
                        ${DOCKER_IMAGE}:${DOCKER_TAG}
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker stop flask-app || true
                    docker rm flask-app || true
                    docker run -d \
                        --name flask-app \
                        --restart always \
                        -p 5000:5000 \
                        ${DOCKER_IMAGE}:${DOCKER_TAG}
                '''
                echo "✅ App running at port 5000"
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline passed! Image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
        }
        failure {
            echo "❌ Pipeline failed. Check stage logs."
        }
        always {
            sh 'docker logout || true'
        }
    }
}
