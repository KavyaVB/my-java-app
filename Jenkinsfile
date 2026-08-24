pipeline {

    agent any

    environment {
        AWS_REGION    = 'us-east-1'
        ECR_REPO_NAME = 'my-java-app'
        ECR_REGISTRY  = 'public.ecr.aws/g1c4x6s2'

        HELM_CHART    = './helm/my-java-app'
        HELM_RELEASE  = 'my-java-app'
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

                    jar tf target/my-java-app-1.0.jar \
                        | grep 'com/example/App.class'

                    echo "Checking MANIFEST..."

                    unzip -p target/my-java-app-1.0.jar \
                        META-INF/MANIFEST.MF

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

        stage('Helm Upgrade') {

            steps {

                withCredentials([
                    file(
                        credentialsId: 'kubernetes_credentials',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        export KUBECONFIG="$KUBECONFIG"

                        echo "======================================"
                        echo "Checking Helm"
                        echo "======================================"

                        helm version

                        echo "======================================"
                        echo "Helm Lint"
                        echo "======================================"

                        helm lint ${HELM_CHART}

                        echo "======================================"
                        echo "Upgrading Application"
                        echo "======================================"

                        helm upgrade --install \
                          ${HELM_RELEASE} \
                          ${HELM_CHART} \
                          --set image.repository="${ECR_REGISTRY}/${ECR_REPO_NAME}" \
                          --set image.tag="${BUILD_NUMBER}" \
                          --wait \
                          --timeout 5m

                        echo "======================================"
                        echo "Helm Upgrade Completed"
                        echo "======================================"

                        helm status ${HELM_RELEASE}
                    '''
                }
            }
        }

        stage('Verify Rolling Update') {

            steps {

                withCredentials([
                    file(
                        credentialsId: 'kubernetes_credentials',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        export KUBECONFIG="$KUBECONFIG"

                        echo "======================================"
                        echo "ROLLING UPDATE STATUS"
                        echo "======================================"

                        kubectl rollout status \
                          deployment/my-java-app \
                          --timeout=5m

                        echo "======================================"
                        echo "Deployment"
                        echo "======================================"

                        kubectl get deployment my-java-app -o wide

                        echo "======================================"
                        echo "Pods"
                        echo "======================================"

                        kubectl get pods -o wide

                        echo "======================================"
                        echo "Service"
                        echo "======================================"

                        kubectl get service
                    '''
                }
            }
        }

        stage('Verify Application Image') {

            steps {

                withCredentials([
                    file(
                        credentialsId: 'kubernetes_credentials',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        export KUBECONFIG="$KUBECONFIG"

                        echo "======================================"
                        echo "DEPLOYED IMAGE"
                        echo "======================================"

                        kubectl get deployment my-java-app \
                          -o jsonpath='{.spec.template.spec.containers[0].image}'

                        echo

                        echo "======================================"
                        echo "READY REPLICAS"
                        echo "======================================"

                        kubectl get deployment my-java-app \
                          -o jsonpath='{.status.readyReplicas}'

                        echo

                        echo "======================================"
                        echo "AVAILABLE REPLICAS"
                        echo "======================================"

                        kubectl get deployment my-java-app \
                          -o jsonpath='{.status.availableReplicas}'

                        echo
                    '''
                }
            }
        }
    }

    post {

        success {

            echo "======================================"
            echo "ROLLING UPDATE SUCCESSFUL"
            echo "======================================"

            echo "Helm Release : ${env.HELM_RELEASE}"

            echo "Image:"
            echo "${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}"
        }

        failure {

            echo "======================================"
            echo "DEPLOYMENT FAILED"
            echo "======================================"

            script {

                withCredentials([
                    file(
                        credentialsId: 'kubernetes_credentials',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        export KUBECONFIG="$KUBECONFIG"

                        echo "======================================"
                        echo "Helm History"
                        echo "======================================"

                        helm history ${HELM_RELEASE}

                        echo "======================================"
                        echo "Current Deployment"
                        echo "======================================"

                        kubectl get deployment my-java-app \
                          -o wide || true

                        echo "======================================"
                        echo "Pod Status"
                        echo "======================================"

                        kubectl get pods -o wide || true

                        echo "======================================"
                        echo "Recent Events"
                        echo "======================================"

                        kubectl get events \
                          --sort-by=.lastTimestamp | tail -20 || true

                        echo "======================================"
                        echo "Rolling Back Helm Release"
                        echo "======================================"

                        helm rollback ${HELM_RELEASE} 0 \
                          --wait \
                          --timeout 5m || true

                        echo "======================================"
                        echo "Rollback Attempt Completed"
                        echo "======================================"
                    '''
                }
            }
        }
    }
}
