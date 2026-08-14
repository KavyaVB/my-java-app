pipeline {

    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPO_NAME = 'my-java-app'
        ECR_REGISTRY = 'public.ecr.aws/g1c4x6s2/java-image'
        IMAGE_NAME = 'java-image'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/KavyaVB/my-java-app.git'
            }
        }

        stage('Build Java') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t ${ECR_REGISTRY}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Login to ECR Public') {
            steps {
                sh '''
                    aws ecr-public get-login-password \
                    --region ${AWS_REGION} | \
                    docker login \
                    --username AWS \
                    --password-stdin public.ecr.aws
                '''
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    docker push \
                    ${ECR_REGISTRY}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        export KUBECONFIG=$KUBECONFIG

                        sed -i \
                        "s|IMAGE_PLACEHOLDER|${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}|g" \
                        k8s/deployment.yaml

                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml

                        kubectl rollout status \
                        deployment/my-java-app
                    '''
                }
            }
        }

        stage('Verify') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        kubectl get pods
                        kubectl get svc
                        kubectl get deployment
                    '''
                }
            }
        }
    }
}
