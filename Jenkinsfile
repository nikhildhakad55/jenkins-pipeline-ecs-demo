pipeline {
    agent any

    environment {
        ECR_REGISTRY = 'account id .dkr.ecr.eu-west-1.amazonaws.com'
        ECR_REPOSITORY = 'give ecr name'
        AWS_DEFAULT_REGION = 'give region name'
        DOCKER_IMAGE = "${ECR_REGISTRY}/${ECR_REPOSITORY}:latest"
        ECS_CLUSTER = 'give name'
        ECS_SERVICE = 'give name'
        ECS_TASK_DEFINITION_FAMILY = 'give name'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git url: 'https://github.com/nikhildhakad55/jenkins-pipeline-ecs-demo.git', branch: 'main'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE} ."
                }
            }
        }

        stage('Login to AWS ECR') {
            steps {
                withCredentials([aws(credentialsId: 'credential id of aws', region: "${AWS_DEFAULT_REGION}")]) {
                    script {
                        sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    sh "docker push ${DOCKER_IMAGE}"
                }
            }
        }

        stage('Update ECS Task Definition') {
            steps {
                withCredentials([aws(credentialsId: 'credential id of aws', region: "${AWS_DEFAULT_REGION}")]) {
                    script {
                        // Fetch the current task definition JSON
                        def taskDefinition = sh(script: "aws ecs describe-task-definition --task-definition ${ECS_TASK_DEFINITION_FAMILY} --query 'taskDefinition' --output json", returnStdout: true).trim()

                        // Use jq to filter out unsupported fields
                        sh """
                        echo '${taskDefinition}' | jq '
                        {
                            family: .family,
                            taskRoleArn: .taskRoleArn,
                            executionRoleArn: .executionRoleArn,
                            networkMode: .networkMode,
                            containerDefinitions: .containerDefinitions,
                            volumes: .volumes,
                            placementConstraints: .placementConstraints,
                            requiresCompatibilities: .requiresCompatibilities,
                            cpu: .cpu,
                            memory: .memory
                        }' > updated-task-definition.json
                        """

                        // Update image field
                        sh "jq '.containerDefinitions[0].image = \"${DOCKER_IMAGE}\"' updated-task-definition.json > new-task-definition.json"

                        // Register the new task definition
                        sh "aws ecs register-task-definition --cli-input-json file://new-task-definition.json"
                    }
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                withCredentials([aws(credentialsId: 'credential id of aws ', region: "${AWS_DEFAULT_REGION}")]) {
                    script {
                        sh "aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --force-new-deployment"
                    }
                }
            }
        }
    }

    post {
        always {
            sh "docker system prune -af"
        }
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed.'
        }
    }
}
