pipeline {
    agent any

    environment {
        APP_NAME = "go-web-app"
        REGISTRY = "gcr.io/applied-pager-476808-j5/${APP_NAME}"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        GCP_SERVICE_ACCOUNT = credentials('gcp-service-account-key') // Jenkins credential ID
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/TCRDINSEH/go-web-app-devops.git'
            }
        }

        stage('Go Build & Test') {
            steps {
                sh '''
                    go mod tidy
                    go build -o ${APP_NAME}
                    go test ./... -v
                '''
            }
        }

        stage('Static Analysis - SonarQube') {
            environment {
                SONARQUBE = credentials('sq-token')
            }
            steps {
                withSonarQubeEnv('SonarQubeServer') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=${APP_NAME} \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=$SONAR_HOST_URL \
                          -Dsonar.login=$SONARQUBE
                    '''
                }
            }
        }

        stage('Dependency Scan') {
            steps {
                sh '''
                    docker run --rm \
                      -v $(pwd):/src \
                      owasp/dependency-check \
                      --scan /src --format HTML --out /src/reports
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${REGISTRY}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Push to Artifact Registry') {
            steps {
                sh '''
                    echo $GCP_SERVICE_ACCOUNT | docker login -u _json_key --password-stdin https://gcr.io
                    docker push ${REGISTRY}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy to GKE via Helm') {
            steps {
                sh '''
                    helm upgrade --install ${APP_NAME} ./helm-chart \
                      --set image.repository=${REGISTRY} \
                      --set image.tag=${IMAGE_TAG}
                '''
            }
        }
    

    stage('Archive Reports') {
    steps {
        archiveArtifacts artifacts: 'reports/*.html', fingerprint: true
    }

    }
    
    }   
}
