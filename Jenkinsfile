pipeline {

    agent any

    environment {
        AWS_REGION    = 'us-east-1'
        ECR_REPO_NAME = 'my-java-app'
        ECR_REGISTRY  = 'public.ecr.aws/g1c4x6s2'

        HELM_CHART = './helm/my-java-app'

        SERVICE_RELEASE = 'my-java-app-service'
        SERVICE_NAME    = 'my-java-app-service'
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

        /*
         * ---------------------------------------------------------
         * Find which color is currently receiving production traffic
         * ---------------------------------------------------------
         */

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

                                kubectl get service ${SERVICE_NAME} \
                                  -o jsonpath='{.spec.selector.version}' \
                                  2>/dev/null || true
                            ''',
                            returnStdout: true
                        ).trim()

                        /*
                         * First deployment.
                         */
                        if (!activeColor) {

                            activeColor = "blue"

                            echo "Production Service does not exist."
                            echo "First deployment will use BLUE."
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

        /*
         * ---------------------------------------------------------
         * Make sure production Service exists.
         * This is a SEPARATE Helm release.
         * ---------------------------------------------------------
         */

        stage('Ensure Production Service') {

            steps {

                withCredentials([
                    file(
                        credentialsId: 'kubernetes_credentials',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        export KUBECONFIG="$KUBECONFIG"

                        echo "Checking production Service..."

                        if kubectl get service ${SERVICE_NAME} >/dev/null 2>&1
                        then
                            echo "Production Service already exists."

                        else

                            echo "Creating production Service..."

                            helm upgrade --install \
                              ${SERVICE_RELEASE} \
                              ${HELM_CHART} \
                              --set productionService.enabled=true \
                              --set productionService.color=${ACTIVE_COLOR} \
                              --wait \
                              --timeout 5m

                            echo "Production Service created."
                        fi

                        kubectl get service ${SERVICE_NAME}
                    '''
                }
            }
        }

        /*
         * ---------------------------------------------------------
         * Deploy GREEN or BLUE without creating a Service
         * ---------------------------------------------------------
         */

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
                        echo "Deploying ${NEW_COLOR}"
                        echo "Image:"
                        echo "${ECR_REGISTRY}/${ECR_REPO_NAME}:${BUILD_NUMBER}"
                        echo "======================================"

                        helm upgrade --install \
                          my-java-app-${NEW_COLOR} \
                          ${HELM_CHART} \
                          --set color=${NEW_COLOR} \
                          --set image.repository="${ECR_REGISTRY}/${ECR_REPO_NAME}" \
                          --set image.tag="${BUILD_NUMBER}" \
                          --set productionService.enabled=false \
                          --wait \
                          --timeout 5m

                        echo "======================================"
                        echo "${NEW_COLOR} deployment completed"
                        echo "======================================"

                        helm status my-java-app-${NEW_COLOR}
                    '''
                }
            }
        }

        /*
         * ---------------------------------------------------------
         * Verify new Deployment
         * ---------------------------------------------------------
         */

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

                        echo "===== Waiting for Pods ====="

                        kubectl wait \
                          --for=condition=ready \
                          pod \
                          -l app=my-java-app,version=${NEW_COLOR} \
                          --timeout=5m
                    '''
                }
            }
        }

        /*
         * ---------------------------------------------------------
         * Switch production traffic
         * ---------------------------------------------------------
         */

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

                        helm upgrade ${SERVICE_RELEASE} \
                          ${HELM_CHART} \
                          --set productionService.enabled=true \
                          --set productionService.color=${NEW_COLOR} \
                          --wait \
                          --timeout 5m

                        echo "Traffic switched to ${NEW_COLOR}."
                    '''
                }
            }
        }

        /*
         * ---------------------------------------------------------
         * Verify Service after traffic switch
         * ---------------------------------------------------------
         */

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

                        kubectl get service ${SERVICE_NAME} \
                          -o jsonpath='{.spec.selector}'
                        echo

                        echo "===== Waiting for Endpoints ====="

                        for i in {1..30}
                        do

                            ENDPOINTS=$(kubectl get endpoints ${SERVICE_NAME} \
                              -o jsonpath='{.subsets[*].addresses[*].ip}' \
                              2>/dev/null || true)

                            if [ -n "$ENDPOINTS" ]
                            then

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

        /*
         * ---------------------------------------------------------
         * Final verification
         * ---------------------------------------------------------
         */

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

                        echo "===== Production Service ====="

                        kubectl get service ${SERVICE_NAME}

                        echo "===== Active Color ====="

                        kubectl get service ${SERVICE_NAME} \
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

    /*
     * -------------------------------------------------------------
     * POST ACTIONS
     * -------------------------------------------------------------
     */

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

                if (env.ACTIVE_COLOR &&
                    env.NEW_COLOR &&
                    env.ACTIVE_COLOR != env.NEW_COLOR) {

                    withCredentials([
                        file(
                            credentialsId: 'kubernetes_credentials',
                            variable: 'KUBECONFIG'
                        )
                    ]) {

                        sh '''
                            export KUBECONFIG="$KUBECONFIG"

                            echo "Rolling production traffic back to:"
                            echo "${ACTIVE_COLOR}"

                            if kubectl get service ${SERVICE_NAME} >/dev/null 2>&1
                            then

                                helm upgrade ${SERVICE_RELEASE} \
                                  ${HELM_CHART} \
                                  --set productionService.enabled=true \
                                  --set productionService.color=${ACTIVE_COLOR} \
                                  --wait \
                                  --timeout 5m

                                echo "======================================"
                                echo "ROLLBACK COMPLETED"
                                echo "======================================"

                                echo "Active color after rollback:"

                                kubectl get service ${SERVICE_NAME} \
                                  -o jsonpath='{.spec.selector.version}'

                                echo

                            else

                                echo "Production Service does not exist."
                                echo "Nothing to rollback."
                            fi
                        '''
                    }
                }
            }
        }
    }
}
