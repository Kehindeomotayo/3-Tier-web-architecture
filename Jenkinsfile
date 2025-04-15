pipeline {
    agent any

    environment {
        FRONTEND_IMAGE = "frontend-app:latest"
        DOCKER_IMAGE = "frontend-app:latest"
        ECR_REPO = "971422716815.dkr.ecr.eu-west-2.amazonaws.com/frontend"
        ECS_TASK_DEFINITION = "webapp-task"
        ECS_CLUSTER = "web-app"
        ECS_SERVICE = "web-service"
        AWS_REGION = "eu-west-2"
        SONAR_PROJECT_KEY = "web-app1"
        SONAR_ORG = "3-tier-wep-app"
        SONAR_SCANNER_PATH = '/opt/sonar-scanner/bin'
        CONTAINER_NAME = "frontend"
        PATH = "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/snap/bin:${SONAR_SCANNER_PATH}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Kehindeomotayo/3-Tier-web-architecture.git'
            }
        }

        stage('SonarCloud Analysis') {
            environment {
                SONAR_TOKEN = credentials('sonar-token')
            }
            steps {
                withSonarQubeEnv('SonarCloud') {
                    sh '''
                    sonar-scanner \
                      -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                      -Dsonar.organization=${SONAR_ORG} \
                      -Dsonar.login=${SONAR_TOKEN} \
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
                sh 'docker build -t ${FRONTEND_IMAGE} -f frontend/Dockerfile frontend/'
            }
        }

        stage('Run Trivy Scan') {
            steps {
                sh 'trivy image --severity HIGH,CRITICAL ${FRONTEND_IMAGE} || exit 1'
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                withCredentials([
                    [$class: 'UsernamePasswordMultiBinding',
                     credentialsId: 'aws-credentials',
                     usernameVariable: 'AWS_ACCESS_KEY_ID',
                     passwordVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {
                    sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                    docker tag ${FRONTEND_IMAGE} ${ECR_REPO}:${BUILD_NUMBER}
                    docker tag ${FRONTEND_IMAGE} ${ECR_REPO}:latest
                    docker push ${ECR_REPO}:${BUILD_NUMBER}
                    docker push ${ECR_REPO}:latest
                    '''
                }
            }
        }

        stage('Update ECS Service') {
            steps {
                withCredentials([
                    [$class: 'UsernamePasswordMultiBinding',
                     credentialsId: 'aws-credentials',
                     usernameVariable: 'AWS_ACCESS_KEY_ID',
                     passwordVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {
                    script {
                        def ecsTaskDefinitionJson = sh(script: "aws ecs describe-task-definition --task-definition ${ECS_TASK_DEFINITION} --output json", returnStdout: true).trim()

                        // Update image inside container definitions
                        def updatedContainerDefs = sh(script: """
                            echo '${ecsTaskDefinitionJson}' | jq '.taskDefinition.containerDefinitions | map(if .name == "${CONTAINER_NAME}" then .image = "${ECR_REPO}:${BUILD_NUMBER}" else . end)' > container-defs.json
                        """, returnStdout: true)

                        def newTaskDef = sh(script: """
                            aws ecs register-task-definition \
                                --family ${ECS_TASK_DEFINITION} \
                                --container-definitions file://container-defs.json \
                                --requires-compatibilities FARGATE \
                                --network-mode awsvpc \
                                --cpu 1024 \
                                --memory 3072 \
                                --execution-role-arn arn:aws:iam::971422716815:role/ecsTaskExecutionRole \
                                --output json
                        """, returnStdout: true).trim()

                        def newRevision = sh(script: "echo '${newTaskDef}' | jq -r '.taskDefinition.revision'", returnStdout: true).trim()

                        // Update ECS service with the new task definition revision
                        sh """
                        aws ecs update-service \
                            --cluster ${ECS_CLUSTER} \
                            --service ${ECS_SERVICE} \
                            --task-definition ${ECS_TASK_DEFINITION}:${newRevision} \
                            --desired-count 1
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "✅ Deployment successful! Image: ${ECR_REPO}:${BUILD_NUMBER}"
        }
        failure {
            echo "❌ Deployment failed at stage: ${env.STAGE_NAME}"
        }
    }
}