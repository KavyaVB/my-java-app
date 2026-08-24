pipeline {

    agent any

    environment {
        AWS_REGION    = 'us-east-1'
        ECR_REPO_NAME = 'my-java-app'
        ECR_REGISTRY  = 'public.ecr.aws/g1c4x6s2'

        HELM_CHART    = './helm/my-java-app'
        HELM_RELEASE  = 'my-java-app'

        DEPLOYMENT_NAME = 'my-java-app'
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
                    set -e

                    echo "======================================"
                    echo "BUILDING JAVA APPLICATION"
                    echo "======================================"

                    mvn clean package
                '''
            }
        }

        stage('Verify JAR') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "VERIFYING JAR"
                    echo "======================================"

                    JAR_FILE="target/my-java-app-1.0.jar"

                    if [ ! -f "$JAR_FILE" ]; then
                        echo "ERROR: JAR file not found: $JAR_FILE"
                        exit 1
                    fi

                    echo "Checking JAR contents..."

                    jar tf "$JAR_FILE" | grep 'com/example/App.class'

                    echo "Checking MANIFEST..."

                    unzip -p "$JAR_FILE" META-INF/MANIFEST.MF

                    echo "JAR verification successful."
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "RUNNING TESTS"
                    echo "======================================"

                    mvn test
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "BUILDING DOCKER IMAGE"
                    echo "======================================"

                    docker build \
                      -t ${ECR_REPO_NAME}:${BUILD_NUMBER} .

                    echo "Docker image built successfully."
                '''
            }
        }

        stage('Login to ECR Public') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "LOGIN TO ECR PUBLIC"
                    echo "======================================"

                    aws ecr-public get-login-password \
                      --region ${AWS_REGION} | \
                    docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}

                    echo "ECR login successful."
                '''
            }
        }

        stage('Tag Image') {
            steps {
                sh '''
                    set -e

                    docker tag \
                      ${ECR_REPO_NAME}:${BUILD_NUMBER} \
                      ${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "PUSHING IMAGE TO ECR"
                    echo "======================================"

                    docker push \
                      ${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}

                    echo "Image pushed successfully."
                    echo "Image: ${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}"
                '''
            }
        }

        stage('Helm Lint') {
            steps {
                sh '''
                    set -e

                    echo "======================================"
                    echo "HELM LINT"
                    echo "======================================"

                    helm version
                    helm lint ${HELM_CHART}
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
                        set -e

                        export KUBECONFIG="$KUBECONFIG"

                        echo "======================================"
                        echo "KUBERNETES CLUSTER"
                        echo "======================================"

                        kubectl cluster-info

                        echo "======================================"
                        echo "HELM UPGRADE"
                        echo "======================================"

                        echo "Helm Release : ${HELM_RELEASE}"
                        echo "Helm Chart   : ${HELM_CHART}"
                        echo "Image        : ${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}"

                        helm upgrade --install \
                          ${HELM_RELEASE} \
                          ${HELM_CHART} \
                          --set image.repository="${ECR_REGISTRY}/${ECR_REPO_NAME}" \
                          --set image.tag="${BUILD_NUMBER}" \
                          --wait \
                          --timeout 5m

                        echo "======================================"
                        echo "HELM RELEASE STATUS"
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
                        set -e

                        export KUBECONFIG="$KUBECONFIG"

                        echo "======================================"
                        echo "ROLLING UPDATE"
                        echo "======================================"

                        echo "Deployment:"
                        kubectl get deployment ${DEPLOYMENT_NAME} -o wide

                        echo "--------------------------------------"
                        echo "Rolling out new version..."
                        echo "--------------------------------------"

                        kubectl rollout status \
                          deployment/${DEPLOYMENT_NAME} \
                          --timeout=5m

                        echo "======================================"
                        echo "ROLLING UPDATE COMPLETED"
                        echo "======================================"
                    '''
                }
            }
        }

        stage('Verify Application') {

            steps {

                withCredentials([
                    file(
                        credentialsId: 'kubernetes_credentials',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        set -e

                        export KUBECONFIG="$KUBECONFIG"

                        echo "======================================"
                        echo "DEPLOYMENT"
                        echo "======================================"

                        kubectl get deployment ${DEPLOYMENT_NAME} -o wide

                        echo "======================================"
                        echo "PODS"
                        echo "======================================"

                        kubectl get pods -o wide

                        echo "======================================"
                        echo "SERVICE"
                        echo "======================================"

                        kubectl get service

                        echo "======================================"
                        echo "DEPLOYED IMAGE"
                        echo "======================================"

                        DEPLOYED_IMAGE=$(kubectl get deployment ${DEPLOYMENT_NAME} \
                          -o jsonpath='{.spec.template.spec.containers[0].image}')

                        echo "Deployed image:"
                        echo "$DEPLOYED_IMAGE"

                        EXPECTED_IMAGE="${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}"

                        echo "Expected image:"
                        echo "$EXPECTED_IMAGE"

                        if [ "$DEPLOYED_IMAGE" != "$EXPECTED_IMAGE" ]; then
                            echo "ERROR: Deployed image does not match expected image."
                            exit 1
                        fi

                        echo "Image verification successful."

                        echo "======================================"
                        echo "READY REPLICAS"
                        echo "======================================"

                        READY_REPLICAS=$(kubectl get deployment ${DEPLOYMENT_NAME} \
                          -o jsonpath='{.status.readyReplicas}')

                        echo "Ready replicas: ${READY_REPLICAS}"

                        echo "======================================"
                        echo "AVAILABLE REPLICAS"
                        echo "======================================"

                        AVAILABLE_REPLICAS=$(kubectl get deployment ${DEPLOYMENT_NAME} \
                          -o jsonpath='{.status.availableReplicas}')

                        echo "Available replicas: ${AVAILABLE_REPLICAS}"
                    '''
                }
            }
        }

        stage('Final Verification') {

            steps {

                withCredentials([
                    file(
                        credentialsId: 'kubernetes_credentials',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        set -e

                        export KUBECONFIG="$KUBECONFIG"

                        echo "======================================"
                        echo "FINAL VERIFICATION"
                        echo "======================================"

                        echo "===== Helm Release ====="

                        helm status ${HELM_RELEASE}

                        echo "===== Helm History ====="

                        helm history ${HELM_RELEASE}

                        echo "===== Deployment ====="

                        kubectl get deployment ${DEPLOYMENT_NAME}

                        echo "===== Pods ====="

                        kubectl get pods -o wide

                        echo "===== Services ====="

                        kubectl get service

                        echo "======================================"
                        echo "ROLLING UPDATE SUCCESSFUL"
                        echo "======================================"
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
            echo "Deployment   : ${env.DEPLOYMENT_NAME}"
            echo "Image        : ${env.ECR_REGISTRY}/${env.ECR_REPO_NAME}:${env.BUILD_NUMBER}"
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
                        echo "HELM HISTORY"
                        echo "======================================"

                        helm history ${HELM_RELEASE} || true

                        echo "======================================"
                        echo "DEPLOYMENT STATUS"
                        echo "======================================"

                        kubectl get deployment ${DEPLOYMENT_NAME} -o wide || true

                        echo "======================================"
                        echo "ROLLING UPDATE STATUS"
                        echo "======================================"

                        kubectl rollout status \
                          deployment/${DEPLOYMENT_NAME} \
                          --timeout=30s || true

                        echo "======================================"
                        echo "PODS"
                        echo "======================================"

                        kubectl get pods -o wide || true

                        echo "======================================"
                        echo "RECENT EVENTS"
                        echo "======================================"

                        kubectl get events \
                          --sort-by=.lastTimestamp | tail -20 || true

                        echo "======================================"
                        echo "DEPLOYMENT FAILED"
                        echo "NO AUTOMATIC HELM ROLLBACK"
                        echo "======================================"

                        echo "Use 'helm history ${HELM_RELEASE}'"
                        echo "to identify the previous successful revision."
                    '''
                }
            }
        }
    }
}
