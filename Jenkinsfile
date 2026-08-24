pipeline {

    agent any

    environment {
        AWS_REGION   = 'us-east-1'
        ECR_REPO_NAME = 'my-java-app'
        ECR_REGISTRY = 'public.ecr.aws/g1c4x6s2'

        // Your Helm chart is inside the my-java-app directory
        HELM_CHART = './helm/my-java-app'
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

        stage('Detect Active Color') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes_credentials',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    script {

                        def activeColor = sh(
                            script: '''
                                export KUBECONFIG="$KUBECONFIG"

                                kubectl get service my-java-app-service \
                                  -o jsonpath='{.spec.selector.version}' \
                                  2>/dev/null || true
                            ''',
                            returnStdout: true
                        ).trim()

                        /*
                         * First deployment:
                         * If the Service does not exist or has no
                         * version selector, start with Blue.
                         */
                        if (!activeColor) {
                            activeColor = "blue"
                        }

                        def newColor

                        if (activeColor == "blue") {
                            newColor = "green"
                        } else {
                            newColor = "blue"
                        }

                        env.ACTIVE_COLOR = activeColor
                        env.NEW_COLOR = newColor

                        echo "======================================"
                        echo "Active color : ${env.ACTIVE_COLOR}"
                        echo "New color    : ${env.NEW_COLOR}"
                        echo "======================================"
                    }
                }
            }
        }

        stage('Deploy Inactive Color') {
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
                        echo "Deploying ${NEW_COLOR} environment"
                        echo "Image:"
                        echo "${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}"
                        echo "======================================"

                        helm upgrade --install \
                          my-java-app-${NEW_COLOR} \
                          ${HELM_CHART} \
                          --set color=${NEW_COLOR} \
                          --set image.repository="${ECR_REGISTRY}/${ECR_REPO_NAME}" \
                          --set image.tag="${BUILD_NUMBER}" \
                          --wait \
                          --timeout 5m

                        echo "Helm deployment completed."

                        helm status my-java-app-${NEW_COLOR}
                    '''
                }
            }
        }

        stage('Verify New Deployment') {
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
                        echo "Checking ${NEW_COLOR} deployment"
                        echo "======================================"

                        kubectl rollout status \
                          deployment/my-java-app-${NEW_COLOR} \
                          --timeout=5m

                        echo "===== Deployment ====="

                        kubectl get deployment \
                          my-java-app-${NEW_COLOR}

                        echo "===== Pods ====="

                        kubectl get pods \
                          -l app=my-java-app,version=${NEW_COLOR} \
                          -o wide

                        echo "===== Pod Readiness ====="

                        kubectl wait \
                          --for=condition=ready \
                          pod \
                          -l app=my-java-app,version=${NEW_COLOR} \
                          --timeout=5m
                    '''
                }
            }
        }

        stage('Switch Traffic') {
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
                        echo "SWITCHING TRAFFIC"
                        echo "FROM: ${ACTIVE_COLOR}"
                        echo "TO  : ${NEW_COLOR}"
                        echo "======================================"

                        kubectl patch service my-java-app-service \
                          -p "{\"spec\":{\"selector\":{\"app\":\"my-java-app\",\"version\":\"${NEW_COLOR}\"}}}"

                        echo "Traffic switched to ${NEW_COLOR}."
                    '''
                }
            }
        }

        stage('Verify Service') {
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
                        echo "VERIFYING PRODUCTION SERVICE"
                        echo "======================================"

                        echo "===== Service Selector ====="

                        kubectl get service my-java-app-service \
                          -o jsonpath='{.spec.selector}'
                        echo

                        echo "===== Service Endpoints ====="

                        for i in {1..30}
                        do

                            ENDPOINTS=$(kubectl get endpoints my-java-app-service \
                              -o jsonpath='{.subsets[*].addresses[*].ip}' \
                              2>/dev/null || true)

                            if [ -n "$ENDPOINTS" ]; then

                                echo "Service endpoints found:"
                                echo "$ENDPOINTS"

                                exit 0
                            fi

                            echo "Waiting for service endpoints..."
                            sleep 5

                        done

                        echo "ERROR: No service endpoints found."

                        exit 1
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
                        export KUBECONFIG="$KUBECONFIG"

                        echo "======================================"
                        echo "FINAL BLUE-GREEN STATUS"
                        echo "======================================"

                        echo "===== Deployments ====="

                        kubectl get deployments

                        echo "===== Pods ====="

                        kubectl get pods -o wide

                        echo "===== Service ====="

                        kubectl get service my-java-app-service

                        echo "===== Active Color ====="

                        kubectl get service my-java-app-service \
                          -o jsonpath='{.spec.selector.version}'
                        echo

                        echo "===== Active Image ====="

                        kubectl get deployment \
                          my-java-app-${NEW_COLOR} \
                          -o jsonpath='{.spec.template.spec.containers[0].image}'
                        echo
                    '''
                }
            }
        }
    }

    post {

        success {

            echo "======================================"
            echo "BLUE-GREEN DEPLOYMENT SUCCESSFUL"
            echo "======================================"

            echo "Previous color : ${env.ACTIVE_COLOR}"
            echo "Active color   : ${env.NEW_COLOR}"

            echo "Image:"
            echo "${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}"
        }

        failure {

            echo "======================================"
            echo "DEPLOYMENT FAILED"
            echo "STARTING AUTOMATIC ROLLBACK"
            echo "======================================"

            script {

                if (env.ACTIVE_COLOR && env.NEW_COLOR) {

                    withCredentials([
                        file(
                            credentialsId: 'kubernetes_credentials',
                            variable: 'KUBECONFIG'
                        )
                    ]) {

                        sh '''
                            export KUBECONFIG="$KUBECONFIG"

                            echo "Rolling traffic back to:"
                            echo "${ACTIVE_COLOR}"

                            kubectl patch service my-java-app-service \
                              -p "{\"spec\":{\"selector\":{\"app\":\"my-java-app\",\"version\":\"${ACTIVE_COLOR}\"}}}"

                            echo "======================================"
                            echo "ROLLBACK COMPLETED"
                            echo "======================================"

                            echo "Active color after rollback:"

                            kubectl get service my-java-app-service \
                              -o jsonpath='{.spec.selector.version}'

                            echo
                        '''
                    }
                }
            }
        }
    }
}
