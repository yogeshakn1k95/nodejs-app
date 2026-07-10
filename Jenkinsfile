pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPOSITORY = 'nodejs-app'
        AWS_ACCOUNT_ID = '280362093954'
        CLUSTER_NAME = 'nodejs-cluster'

        IMAGE_TAG = "${BUILD_NUMBER}"

        IMAGE_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:${IMAGE_TAG}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t ${ECR_REPOSITORY}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials'
                ]]) {

                    sh '''
                    aws ecr get-login-password --region ${AWS_REGION} \
                    | docker login \
                    --username AWS \
                    --password-stdin \
                    ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Push Image to Amazon ECR') {
            steps {

                sh '''
                docker tag ${ECR_REPOSITORY}:${IMAGE_TAG} ${IMAGE_URI}

                docker push ${IMAGE_URI}
                '''
            }
        }

        stage('Configure kubeconfig') {
            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials'
                ]]) {

                    sh '''
                    aws eks update-kubeconfig \
                    --region ${AWS_REGION} \
                    --name ${CLUSTER_NAME}
                    '''
                }
            }
        }

        stage('Deploy to Amazon EKS') {
            steps {

                sh '''
                sed -i "s|IMAGE_PLACEHOLDER|${IMAGE_URI}|g" k8s/deployment.yaml

                kubectl apply -f k8s/deployment.yaml

                kubectl apply -f k8s/service.yaml
                '''
            }
        }

        stage('Verify Rollout') {
            steps {

                sh '''
                kubectl rollout status deployment/nodejs-app

                kubectl get pods

                kubectl get svc
                '''
            }
        }
    }
}
