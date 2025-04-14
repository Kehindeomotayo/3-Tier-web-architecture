pipeline {
    agent any

    environment {
        FRONTEND_IMAGE = "frontend-app:latest"
        DOCKER_IMAGE = "frontend-app:latest"
        AWS_ACCESS_KEY_ID = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
        ECR_REPO = "971422716815.dkr.ecr.eu-west-2.amazonaws.com/frontend"
        ECS_TASK_DEFINITION = "webapp-task"
        ECS_CLUSTER = "web-app"
        ECS_SERVICE = "web-service"
        AWS_REGION = "eu-west-1"
        SONAR_PROJECT_KEY = "wep-app1"
        SONAR_ORG = "3-tier-wep-app"
        SONAR_TOKEN = credentials('sonarcloud-token')
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
                git branch: 'main', url: 'git@github.com:Kehindeomotayo/3-Tier-web-architecture.git'
            }
        }

        stage('SonarCloud Analysis') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    sh '''
                    sonar-scanner \
                    -Dsonar.projectKey=wep-app1 \
                    -Dsonar.organization=3-tier-wep-app \
                    -Dsonar.login=$SONAR_TOKEN \
                    -Dsonar.host.url=https://sonarcloud.io \
                    -Dsonar.sourceEncoding=UTF-8 \
                    -Dsonar.sources=frontend \
                    -Dsonar.exclusions=**/test/**,**/*.spec.js
                    '''
                }
            }
        }

        stage('Wait for Quality Gate') {
            steps {
                script {
                    def qualityGate = waitForQualityGate()
                    if (qualityGate.status != 'OK') {
                        error "Quality gate failed: ${qualityGate.status}"
                    }
                }
            }
        }

        stage('Build Frontend Docker Image') {
            steps {
                sh """
                docker build -t $FRONTEND_IMAGE -f frontend/Dockerfile frontend/
                """
            }
        }

        stage('Run Trivy Scan') {
            steps {
                sh "trivy image --severity HIGH,CRITICAL $FRONTEND_IMAGE || exit 1"
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
                docker tag $FRONTEND_IMAGE $ECR_REPO:$BUILD_NUMBER
                docker push $ECR_REPO:$BUILD_NUMBER
                """
            }
        }

        stage('Update ECS Service') {
            steps {
                script {
                    try {
                        def ecsTaskDefinition = sh(script: "aws ecs describe-task-definition --task-definition $ECS_TASK_DEFINITION", returnStdout: true).trim()

                        def executionRoleArn = "arn:aws:iam::971422716815:role/ecsTaskExecutionRole"

                        def updatedTaskDefinition = sh(script: """
                            echo '$ecsTaskDefinition' | jq -r '.taskDefinition.containerDefinitions | map(if .name == "frontend" then .image = "$ECR_REPO:$BUILD_NUMBER" else . end)' | jq -s '.[0]'
                        """, returnStdout: true).trim()

                        def newTaskDefinition = sh(script: """
                            aws ecs register-task-definition --family $ECS_TASK_DEFINITION \
                                --container-definitions '$updatedTaskDefinition' \
                                --requires-compatibilities FARGATE \
                                --network-mode awsvpc \
                                --cpu 1024 \
                                --memory 3072 \
                                --execution-role-arn $executionRoleArn
                        """, returnStdout: true).trim()

                        def newTaskRevision = sh(script: """
                            echo '$newTaskDefinition' | jq -r '.taskDefinition.revision'
                        """, returnStdout: true).trim()

                        sh """
                        aws ecs update-service \
                            --cluster $ECS_CLUSTER \
                            --service $ECS_SERVICE \
                            --task-definition $ECS_TASK_DEFINITION:$newTaskRevision
                        """
                    } catch (Exception e) {
                        echo "Error during ECS service update: ${e.message}"
                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
