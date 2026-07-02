pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('shown14-dockerhub-password')
        SONAR_TOKEN = credentials('shown14-sonar-token')
        IMAGE_NAME = "shown14/tanguy-tasklist-backend"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Generate Prisma Client') {
            steps {
                sh 'npm run prisma:generate'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mkdir -p reports'
                sh 'npm run test:coverage'
                sh 'cp reports/junit.xml reports/junit-unit.xml'
            }
            post {
                always {
                    junit testResults: 'reports/junit-unit.xml', allowEmptyResults: true
                }
            }
        }

        stage('E2E Tests') {
            steps {
                sh 'npm run test:e2e:coverage'
                sh 'cp reports/junit.xml reports/junit-e2e.xml'
            }
            post {
                always {
                    junit testResults: 'reports/junit-e2e.xml', allowEmptyResults: true
                }
            }
        }

       stage('SonarQube Analysis') {
             steps {
                 script {
                    def scannerHome = tool 'SonarScanner'
                    sh """
                    ${scannerHome}/bin/sonar-scanner \
                  -Dsonar.host.url=https://sonarqube.cicd.kits.ext.educentre.fr \
                  -Dsonar.token=${SONAR_TOKEN} \
                  -Dsonar.projectKey=tanguy-tasklist-backend \
                  -Dsonar.projectName='tanguy - TaskList Backend' \
                  -Dsonar.sources=src \
                  -Dsonar.exclusions=src/__tests__/** \
                  -Dsonar.tests=src/__tests__ \
                  -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                  -Dsonar.qualitygate.wait=true
                """
                }
            }
        }

        stage('Build Docker image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."
            }
        }

       stage('Trivy Security Scan') {
         steps {
            sh 'mkdir -p security-reports'
            sh "trivy image --format table --output security-reports/trivy-report.txt --severity HIGH,CRITICAL --exit-code 0 ${IMAGE_NAME}:${IMAGE_TAG}"
        }
        post {
            always {
                archiveArtifacts artifacts: 'security-reports/*', fingerprint: true
                }
            }
        }

        stage('Generate SBOM') {
            steps {
                sh "trivy image ${IMAGE_NAME}:${IMAGE_TAG} --format spdx-json --output sbom-spdx.json"
            }
            post {
                always {
                    archiveArtifacts artifacts: 'sbom-spdx.json', fingerprint: true
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'shown14-dockerhub-password', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                sh "docker push ${IMAGE_NAME}:latest"
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            cleanWs()
        }
    }
}