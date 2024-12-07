pipeline {
    agent any

    environment {
        FRONTEND_IMAGE = "your-frontend-image:latest"
        DOCKER_IMAGE = "your-image:latest" 
        SONAR_PROJECT_KEY = "scan"
        SONAR_ORG = "3-tier-architecture-project"
        SONAR_TOKEN = credentials('sonar-login') // Add token in Jenkins credentials
        SONAR_SCANNER_PATH = '/opt/sonar-scanner/bin'
        PATH = "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/snap/bin:${SONAR_SCANNER_PATH}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/Kehindeomotayo/3-Tier-web-architecture.git'
            }
        }

        stage('Build Frontend Docker Image') {
            steps {
                script {
                    sh """
                    docker build -t $FRONTEND_IMAGE -f frontend/Dockerfile frontend/
                    """
                }
            }
        }
        stage('SonarCloud Analysis') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    sh '''
                    sonar-scanner \
                    -Dsonar.projectKey=scan \
                    -Dsonar.organization=3-tier-architecture-project \
                    -Dsonar.login=$SONAR_TOKEN \
                    -Dsonar.host.url=https://sonarcloud.io \
                    -Dsonar.sourceEncoding=UTF-8 \
                    -Dsonar.sources=frontend \
                    -Dsonar.exclusions=**/test/**,**/*.spec.js
                    '''
                }
            }
        }
    }
}
