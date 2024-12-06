pipeline {
    agent any

    environment {
        FRONTEND_IMAGE = "your-frontend-image:latest"
        DOCKER_IMAGE = "your-image:latest" 
        SONAR_PROJECT_KEY = "3-tier-architecture-project_scan"
        SONAR_ORG = "3-tier-architecture-project"
        SONAR_TOKEN = credentials('sonar-login') // Add token in Jenkins credentials
        SONAR_SCANNER_PATH = '/opt/sonar-scanner/bin'
        PATH = "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/snap/bin:${SONAR_SCANNER_PATH}"
        ECR_REPO = "971422716815.dkr.ecr.eu-west-1.amazonaws.com/frontend"
        AWS_REGION = "eu-west-1"
        AWS_ACCESS_KEY_ID = credentials('access-key') // Jenkins credential ID for access key
        AWS_SECRET_ACCESS_KEY = credentials('access-key') // Jenkins credential ID for secret key
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
        stage('Run Trivy Scan') {
            steps {
                script {
                    // Scan the Docker image for vulnerabilities
                    sh "trivy image --severity HIGH,CRITICAL $FRONTEND_IMAGE || exit 1"
                }
            }
        }
        stage('Push Docker Image to ECR') {
            steps {
                script {
                    // Authenticate Docker to ECR
                    sh """
                    aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
                    docker tag $FRONTEND_IMAGE $ECR_REPO:$BUILD_NUMBER
                    docker push $ECR_REPO:$BUILD_NUMBER
                    """
                }
            }
        }

    }
}
      