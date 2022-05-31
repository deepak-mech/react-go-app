pipeline {
  environment {
    SERVICE_NAME = 'go-backend'
    TASK_FAMILY="go-backend" // at least one container needs to have the same name as the task definition
    DESIRED_COUNT="1"
    CLUSTER_NAME = "demo-rgp"
    IMAGE_TAG = "${BUILD_NUMBER}"
    AWS_ACCOUNT_ID="897708493501"
    AWS_DEFAULT_REGION="us-east-1" 
    IMAGE_REPO_NAME="go-backend"
    REPOSITORY_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
    EXECUTION_ROLE_ARN = "arn:aws:iam::897708493501:role/ecsTaskExecutionRole"
  }

    agent any

    stages {
        
         stage('Logging into AWS ECR') {
            steps {
                script {
                sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
                }                 
            }
        }
        
        stage('GitSCM') {
            steps {
                git branch: 'feature_1', credentialsId: 'd71e8864-c73b-4350-bac7-c50c63042065', url: 'https://github.com/applied-ai-consulting/C01-sample-react-frontend.git'     
            }
        }
  
    // Building Docker images
    stage('Building image') {
      steps{
        script {
          dockerImage = docker.build "${IMAGE_REPO_NAME}:${IMAGE_TAG}"
        }
      }
    }
   
    // Uploading Docker images into AWS ECR
    stage('Pushing to ECR') {
     steps{  
         script {
                sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:$IMAGE_TAG"
                sh "docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}"
         }
       }
    }
     
    // Deploy Image to ECS
    stage('Deploy Image to ECS') {
            steps{
                // prepare task definition file
                sh """sed -e "s;%REPOSITORY_URI%;${REPOSITORY_URI};g" -e "s;%IMAGE_TAG%;${IMAGE_TAG};g" -e "s;%TASK_FAMILY%;${TASK_FAMILY};g" -e "s;%SERVICE_NAME%;${SERVICE_NAME};g" -e "s;%EXECUTION_ROLE_ARN%;${EXECUTION_ROLE_ARN};g" taskdef_template.json > taskdef_${IMAGE_TAG}.json"""
                script {
                    // Register task definition
                    AWS("ecs register-task-definition --output json --cli-input-json file://${WORKSPACE}/taskdef_${IMAGE_TAG}.json > ${env.WORKSPACE}/temp.json")
                    def projects = readJSON file: "${env.WORKSPACE}/temp.json"
                    def TASK_REVISION = projects.taskDefinition.revision

                    // update service
                    AWS("ecs update-service --cluster ${CLUSTER_NAME} --service ${SERVICE_NAME} --task-definition ${TASK_FAMILY}:${TASK_REVISION} --desired-count ${DESIRED_COUNT}")
                }
            }
        }
     
     // Remove Image
      stage('Remove docker image') {
            steps{
                // Remove images
                sh "docker rmi $REPOSITORY_URI:$IMAGE_TAG"
            }
       } 
   }
}
