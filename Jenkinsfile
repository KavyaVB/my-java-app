pipeline {

    agent any

    environment {
        AWS_REGION   = 'us-east-1'
        ECR_REPO_NAME = 'my-java-app'
        ECR_REGISTRY = 'public.ecr.aws/g1c4x6s2'
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
                sh '''
                    mvn clean package
                '''
            }
        }

        stage('Verify JAR') {
            steps {
                sh '''
                    echo "Checking JAR contents..."
                    jar tf target/my-java-app-1.0.jar | grep 'com/example/App.class'

                    echo "Checking MANIFEST..."
                    unzip -p target/my-java-app-1.0.jar META-INF/MANIFEST.MF

                    echo "Testing application..."
                    timeout 10 java -jar target/my-java-app-1.0.jar || true
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    mvn test
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t ${ECR_REPO_NAME}:${BUILD_NUMBER} .
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
                      --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Tag Image') {
            steps {
                sh '''
                    docker tag \
                      ${ECR_REPO_NAME}:${BUILD_NUMBER} \
                      ${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    docker push \
                      ${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes_credentials',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        export KUBECONFIG="$KUBECONFIG"

                        echo "Deploying image:"
                        echo "${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}"

                        kubectl set image deployment/my-java-app \
                          my-java-app=${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}

                        kubectl apply -f k8s/service.yaml

                        kubectl rollout status \
                          deployment/my-java-app \
                          --timeout=5m
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes_credentials',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        export KUBECONFIG="$KUBECONFIG"

                        echo "===== Deployment ====="
                        kubectl get deployment my-java-app

                        echo "===== Pods ====="
                        kubectl get pods -o wide

                        echo "===== Service ====="
                        kubectl get svc my-java-app-service

                        echo "===== Image ====="
                        kubectl get deployment my-java-app \
                          -o jsonpath='{.spec.template.spec.containers[0].image}'
                        echo
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Build ${BUILD_NUMBER} completed successfully."
            echo "Image: ${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}"
        }

        failure {
            echo "Build ${BUILD_NUMBER} failed."
            echo "Check the failed stage and Jenkins console output."
        }
    }
}
