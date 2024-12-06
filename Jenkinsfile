pipeline {
    agent any

    environment {
        FRONTEND_IMAGE = "your-frontend-image:latest"
        DOCKER_IMAGE = "your-image:latest"
        AWS_ACCESS_KEY_ID = credentials('aws-key') // Jenkins credential ID for access key
        AWS_SECRET_ACCESS_KEY = credentials('aws-key') // Jenkins credential ID for secret key
        ECR_REPO = "429841094792.dkr.ecr.us-east-1.amazonaws.com/frontend"
        ECS_TASK_DEFINITION = "task-web-app"
        ECS_CLUSTER =  "Full-stack-web-app"
        ECS_SERVICE = "web-app-service"
        AWS_REGION = "us-east-1"
        SONAR_PROJECT_KEY = "3-tier-architecture-project_scan"
        SONAR_ORG = "3-Tier-Architecture-project"
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
                    -Dsonar.projectKey=3-tier-architecture-project_scan \
                    -Dsonar.organization=3-Tier-Architecture-project \
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
