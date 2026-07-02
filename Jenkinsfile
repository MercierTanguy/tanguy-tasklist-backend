pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('shown14-dockerhub-password')
        SONAR_TOKEN = credentials('shown14-sonar-token')
        IMAGE_NAME = "shown14/tanguy-tasklist-backend"
        SONAR_HOST_URL = "https://sonarqube.cicd.kits.ext.educentre.fr"
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
                sh 'npm run test:coverage'
            }
        }

        stage('Publish Test Reports') {
            steps {
                junit allowEmptyResults: true, testResults: 'reports/junit.xml'
            }
        }

        stage('E2E Tests') {
            steps {
                sh 'npm run test:e2e:coverage'
            }
        }

       stage('SonarQube Analysis') {
             steps {
                sh '''
                     if [ ! -d "$HOME/sonar-scanner" ]; then
                    curl -sSLo sonar-scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
                    python3 -c "import zipfile; zipfile.ZipFile('sonar-scanner.zip').extractall('$HOME')"
                    mv $HOME/sonar-scanner-* $HOME/sonar-scanner
                fi
            export PATH=$HOME/sonar-scanner/bin:$PATH
            sonar-scanner \
              -Dsonar.host.url=$SONAR_HOST_URL \
              -Dsonar.token=$SONAR_TOKEN \
              -Dsonar.qualitygate.wait=true
            '''
            }
        }

        stage('Build Docker image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh """
                    mkdir -p trivy-reports
                    trivy image --severity CRITICAL,HIGH --exit-code 1 \
                      --format table \
                      --output trivy-reports/trivy-report.txt \
                      ${IMAGE_NAME}:${BUILD_NUMBER}
                """
            }
        }

        stage('Generate SBOM') {
            steps {
                sh """
                    trivy image --format spdx-json \
                      --output sbom-spdx.json \
                      ${IMAGE_NAME}:${BUILD_NUMBER}
                """
            }
        }

        stage('Archive Reports') {
            steps {
                archiveArtifacts artifacts: 'trivy-reports/*.txt', allowEmptyArchive: true
                archiveArtifacts artifacts: 'sbom-spdx.json', allowEmptyArchive: true
            }
        }

        stage('Push to DockerHub') {
            steps {
                sh "echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin"
                sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
                sh "docker push ${IMAGE_NAME}:latest"
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}