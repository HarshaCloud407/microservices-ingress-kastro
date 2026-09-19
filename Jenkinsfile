pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        buildDiscarder(logRotator(
            numToKeepStr: '20',
            artifactNumToKeepStr: '10'
        ))
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        DOCKER_HUB_REPO = 'hariprasad123456/techsolutions-app'

        K8S_CLUSTER_NAME = 'blujay-cluster-hyd2'
        AWS_REGION       = 'us-east-1'
        NAMESPACE        = 'default'

        APP_NAME         = 'techsolutions'
        DEPLOYMENT_NAME  = 'techsolutions-deployment'
        SERVICE_NAME     = 'techsolutions-service'
        INGRESS_NAME     = 'techsolutions-ingress'

        INGRESS_NAMESPACE = 'ingress-nginx'
        INGRESS_SERVICE   = 'ingress-nginx-controller'

        DOCKER_CREDENTIALS = 'dockerhub-credentails'
        AWS_CREDENTIALS    = 'aws-creds'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'

                git(
                    branch: 'master',
                    url: 'https://github.com/HarshaCloud407/microservices-ingress-kastro.git'
                )

                sh '''
                    echo "Git commit:"
                    git rev-parse --short HEAD

                    echo "Git branch:"
                    git branch --show-current
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'

                script {
                    env.IMAGE_TAG = "${BUILD_NUMBER}"
                    env.IMAGE_NAME = "${DOCKER_HUB_REPO}:${env.IMAGE_TAG}"
                    env.LATEST_IMAGE = "${DOCKER_HUB_REPO}:latest"

                    sh '''
                        set -e

                        echo "Building immutable image:"
                        echo "${IMAGE_NAME}"

                        docker build \
                            --pull \
                            -t "${IMAGE_NAME}" \
                            .

                        docker tag "${IMAGE_NAME}" "${LATEST_IMAGE}"

                        echo "Docker images created:"
                        docker images "${DOCKER_HUB_REPO}" --format "table {{.Repository}}\\t{{.Tag}}\\t{{.ID}}\\t{{.Size}}"
                    '''
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo 'Pushing Docker images to Docker Hub...'

                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -e

                        echo "${DOCKER_PASSWORD}" | docker login \
                            --username "${DOCKER_USERNAME}" \
                            --password-stdin

                        docker push "${IMAGE_NAME}"
                        docker push "${LATEST_IMAGE}"

                        docker logout || true

                        echo "Docker images pushed successfully:"
                        echo "${IMAGE_NAME}"
                        echo "${LATEST_IMAGE}"
                    '''
                }
            }
        }

        stage('Configure AWS and Kubectl') {
            steps {
                echo 'Configuring AWS CLI and kubectl...'

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS}"]
                ]) {
                    sh '''
                        set -e

                        aws configure set region "${AWS_REGION}"

                        echo "AWS Account:"
                        aws sts get-caller-identity

                        echo "Updating kubeconfig..."
                        aws eks update-kubeconfig \
                            --region "${AWS_REGION}" \
                            --name "${K8S_CLUSTER_NAME}"

                        echo "Current Kubernetes context:"
                        kubectl config current-context

                        echo "EKS Nodes:"
                        kubectl get nodes -o wide

                        echo "Kubernetes version:"
                        kubectl version --short 2>/dev/null || kubectl version
                    '''
                }
            }
        }

        stage('Validate Kubernetes Resources') {
            steps {
                echo 'Validating existing Kubernetes resources...'

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS}"]
                ]) {
                    sh '''
                        set -e

                        echo "Deployment:"
                        kubectl get deployment "${DEPLOYMENT_NAME}" \
                            -n "${NAMESPACE}" || true

                        echo "Service:"
                        kubectl get service "${SERVICE_NAME}" \
                            -n "${NAMESPACE}" || true

                        echo "Ingress:"
                        kubectl get ingress "${INGRESS_NAME}" \
                            -n "${NAMESPACE}" || true
                    '''
                }
            }
        }

        stage('Deploy Application') {
            steps {
                echo 'Deploying application to Kubernetes...'

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS}"]
                ]) {
                    sh '''
                        set -e

                        echo "Applying Kubernetes manifest..."
                        kubectl apply \
                            -f k8s/deployment.yaml \
                            -n "${NAMESPACE}"

                        echo "Detecting application container name..."

                        CONTAINER_NAME=$(kubectl get deployment "${DEPLOYMENT_NAME}" \
                            -n "${NAMESPACE}" \
                            -o jsonpath='{.spec.template.spec.containers[0].name}')

                        if [ -z "${CONTAINER_NAME}" ]; then
                            echo "ERROR: Could not determine container name."
                            exit 1
                        fi

                        echo "Container name: ${CONTAINER_NAME}"
                        echo "Updating image to: ${IMAGE_NAME}"

                        kubectl set image \
                            deployment/"${DEPLOYMENT_NAME}" \
                            "${CONTAINER_NAME}"="${IMAGE_NAME}" \
                            -n "${NAMESPACE}"

                        echo "Waiting for rollout..."

                        kubectl rollout status \
                            deployment/"${DEPLOYMENT_NAME}" \
                            -n "${NAMESPACE}" \
                            --timeout=300s

                        echo "Deployment status:"
                        kubectl get deployment "${DEPLOYMENT_NAME}" \
                            -n "${NAMESPACE}"

                        echo "Application pods:"
                        kubectl get pods \
                            -n "${NAMESPACE}" \
                            -l "app=${APP_NAME}" \
                            -o wide

                        echo "Application service:"
                        kubectl get service "${SERVICE_NAME}" \
                            -n "${NAMESPACE}"
                    }
                }
            }
        }

        stage('Deploy Ingress') {
            steps {
                echo 'Applying Ingress resource...'

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS}"]
                ]) {
                    sh '''
                        set -e

                        kubectl apply \
                            -f k8s/ingress.yaml \
                            -n "${NAMESPACE}"

                        echo "Ingress resource:"
                        kubectl get ingress "${INGRESS_NAME}" \
                            -n "${NAMESPACE}" \
                            -o wide

                        echo "Ingress details:"
                        kubectl describe ingress "${INGRESS_NAME}" \
                            -n "${NAMESPACE}"
                    }
                }
            }
        }

        stage('Validate NGINX Ingress Controller') {
            steps {
                echo 'Validating NGINX Ingress Controller...'

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS}"]
                ]) {
                    sh '''
                        set -e

                        echo "NGINX controller pods:"
                        kubectl get pods \
                            -n "${INGRESS_NAMESPACE}" \
                            -l app.kubernetes.io/component=controller \
                            -o wide

                        echo "NGINX controller service:"
                        kubectl get svc "${INGRESS_SERVICE}" \
                            -n "${INGRESS_NAMESPACE}" \
                            -o wide

                        echo "NGINX controller endpoints:"
                        kubectl get endpoints "${INGRESS_SERVICE}" \
                            -n "${INGRESS_NAMESPACE}" || true
                    '''
                }
            }
        }

        stage('Get Ingress URL') {
            steps {
                echo 'Waiting for AWS Load Balancer...'

                script {
                    withCredentials([
                        [$class: 'AmazonWebServicesCredentialsBinding',
                         credentialsId: "${AWS_CREDENTIALS}"]
                    ]) {

                        def ingressAddress = ''

                        for (int attempt = 1; attempt <= 24; attempt++) {

                            echo "Checking LoadBalancer status - attempt ${attempt}/24"

                            ingressAddress = sh(
                                script: '''
                                    kubectl get svc "${INGRESS_SERVICE}" \
                                        -n "${INGRESS_NAMESPACE}" \
                                        -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' \
                                        2>/dev/null || true
                                ''',
                                returnStdout: true
                            ).trim()

                            if (!ingressAddress) {
                                ingressAddress = sh(
                                    script: '''
                                        kubectl get svc "${INGRESS_SERVICE}" \
                                            -n "${INGRESS_NAMESPACE}" \
                                            -o jsonpath='{.status.loadBalancer.ingress[0].ip}' \
                                            2>/dev/null || true
                                    ''',
                                    returnStdout: true
                                ).trim()
                            }

                            if (ingressAddress) {
                                env.INGRESS_ADDRESS = ingressAddress
                                env.INGRESS_URL = "http://${ingressAddress}"

                                echo "========================================="
                                echo "AWS LOAD BALANCER CREATED"
                                echo "========================================="
                                echo "Ingress Address: ${env.INGRESS_ADDRESS}"
                                echo "Ingress URL: ${env.INGRESS_URL}"
                                echo "========================================="

                                break
                            }

                            echo "LoadBalancer is still pending."

                            def serviceDetails = sh(
                                script: '''
                                    kubectl get svc "${INGRESS_SERVICE}" \
                                        -n "${INGRESS_NAMESPACE}" \
                                        -o wide || true
                                ''',
                                returnStdout: true
                            ).trim()

                            echo serviceDetails

                            def events = sh(
                                script: '''
                                    kubectl get events \
                                        -n "${INGRESS_NAMESPACE}" \
                                        --sort-by=.lastTimestamp \
                                        2>/dev/null | tail -n 30 || true
                                ''',
                                returnStdout: true
                            ).trim()

                            echo "Recent ingress-nginx events:"
                            echo events

                            if (
                                events.contains('OperationNotPermitted') ||
                                events.contains('does not support creating load balancers') ||
                                events.contains('failed to ensure load balancer') ||
                                events.contains('CreateLoadBalancer')
                            ) {
                                sh '''
                                    echo "=============================================="
                                    echo "AWS LOAD BALANCER CREATION FAILED"
                                    echo "=============================================="

                                    kubectl describe svc "${INGRESS_SERVICE}" \
                                        -n "${INGRESS_NAMESPACE}" || true

                                    echo ""
                                    echo "Recent Kubernetes events:"
                                    kubectl get events \
                                        -n "${INGRESS_NAMESPACE}" \
                                        --sort-by=.lastTimestamp \
                                        | tail -n 50 || true

                                    echo "=============================================="
                                '''

                                error(
                                    "AWS is refusing LoadBalancer creation for " +
                                    "ingress-nginx-controller. " +
                                    "Kubernetes and Jenkins are working, but the AWS " +
                                    "account/ELB service is returning OperationNotPermitted. " +
                                    "Resolve the AWS account restriction before expecting " +
                                    "a public Ingress URL."
                                )
                            }

                            if (attempt < 24) {
                                echo "Waiting 10 seconds before next check..."
                                sleep(time: 10, unit: 'SECONDS')
                            }
                        }

                        if (!env.INGRESS_ADDRESS) {

                            sh '''
                                echo "=============================================="
                                echo "LOAD BALANCER NOT READY"
                                echo "=============================================="

                                kubectl get svc "${INGRESS_SERVICE}" \
                                    -n "${INGRESS_NAMESPACE}" \
                                    -o wide || true

                                kubectl describe svc "${INGRESS_SERVICE}" \
                                    -n "${INGRESS_NAMESPACE}" || true

                                echo ""
                                echo "Recent events:"
                                kubectl get events \
                                    -n "${INGRESS_NAMESPACE}" \
                                    --sort-by=.lastTimestamp \
                                    | tail -n 50 || true

                                echo "=============================================="
                            '''

                            error(
                                "Ingress LoadBalancer did not receive an external " +
                                "hostname/IP within the configured timeout."
                            )
                        }
                    }
                }
            }
        }

        stage('Application Health Check') {
            when {
                expression {
                    return env.INGRESS_URL?.trim()
                }
            }

            steps {
                echo 'Testing application through the Ingress LoadBalancer...'

                script {
                    def paths = [
                        '/',
                        '/about',
                        '/services',
                        '/contact'
                    ]

                    for (String path : paths) {

                        echo "Testing: ${env.INGRESS_URL}${path}"

                        sh """
                            set -e

                            curl \
                                --fail \
                                --silent \
                                --show-error \
                                --connect-timeout 10 \
                                --max-time 30 \
                                -o /dev/null \
                                -w "HTTP Status: %{http_code}\\n" \
                                "${env.INGRESS_URL}${path}"
                        """
                    }
                }
            }
        }

        stage('Final Kubernetes Validation') {
            steps {
                echo 'Final Kubernetes validation...'

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS}"]
                ]) {
                    sh '''
                        set -e

                        echo "================ FINAL STATUS ================"

                        echo ""
                        echo "Nodes:"
                        kubectl get nodes

                        echo ""
                        echo "Application Deployment:"
                        kubectl get deployment \
                            "${DEPLOYMENT_NAME}" \
                            -n "${NAMESPACE}"

                        echo ""
                        echo "Application Pods:"
                        kubectl get pods \
                            -n "${NAMESPACE}" \
                            -l "app=${APP_NAME}" \
                            -o wide

                        echo ""
                        echo "Application Service:"
                        kubectl get svc \
                            "${SERVICE_NAME}" \
                            -n "${NAMESPACE}"

                        echo ""
                        echo "Ingress:"
                        kubectl get ingress \
                            "${INGRESS_NAME}" \
                            -n "${NAMESPACE}" \
                            -o wide

                        echo ""
                        echo "NGINX LoadBalancer:"
                        kubectl get svc \
                            "${INGRESS_SERVICE}" \
                            -n "${INGRESS_NAMESPACE}" \
                            -o wide

                        echo ""
                        echo "================================================"
                    '''
                }
            }
        }
    }

    post {

        always {
            echo 'Collecting final deployment information...'

            sh '''
                echo "========== Kubernetes Diagnostic Information =========="

                kubectl get pods \
                    -n "${NAMESPACE}" \
                    -l "app=${APP_NAME}" \
                    -o wide 2>/dev/null || true

                kubectl get svc \
                    -n "${NAMESPACE}" \
                    2>/dev/null || true

                kubectl get ingress \
                    -n "${NAMESPACE}" \
                    2>/dev/null || true

                kubectl get svc "${INGRESS_SERVICE}" \
                    -n "${INGRESS_NAMESPACE}" \
                    -o wide 2>/dev/null || true

                echo "======================================================="
            '''

            echo 'Cleaning only local Docker images...'

            sh '''
                docker image rm "${IMAGE_NAME}" 2>/dev/null || true
                docker image rm "${LATEST_IMAGE}" 2>/dev/null || true
            '''
        }

        success {
            echo '================================================'
            echo 'PIPELINE COMPLETED SUCCESSFULLY'
            echo '================================================'
            echo "Docker Image: ${env.IMAGE_NAME}"
            echo "Ingress URL:  ${env.INGRESS_URL}"
            echo '================================================'
        }

        failure {
            echo '================================================'
            echo 'PIPELINE FAILED'
            echo '================================================'
            echo 'Check the stage logs above for the exact failure.'
            echo 'No Kubernetes resources were deleted by this pipeline.'
            echo '================================================'
        }

        aborted {
            echo 'Pipeline was aborted.'
            echo 'No Kubernetes resources were deleted by this pipeline.'
        }
    }
}