pipeline {
    agent any

    environment {
        AWS_REGION     = 'ap-south-1'
        ECR_REGISTRY   = '129463260199.dkr.ecr.ap-south-1.amazonaws.com'
        ECR_REPOSITORY = 'application-code'
        ECR_IMAGE      = '129463260199.dkr.ecr.ap-south-1.amazonaws.com/application-code:latest'
        EKS_CLUSTER    = 'my-eks-cluster'
        NAMESPACE      = 'application'
    }

    stages {

        stage('Verify Kubernetes Files') {
            steps {
                sh '''
                    echo "===== Workspace ====="
                    pwd

                    echo "===== Repository Files ====="
                    ls -la

                    echo "===== Kubernetes Manifests ====="

                    test -f namespace.yaml
                    test -f deployment.yaml
                    test -f service.yaml

                    echo "All Kubernetes manifest files found."
                '''
            }
        }

        stage('AWS Authentication') {
            steps {
                sh '''
                    echo "===== AWS Identity ====="
                    aws sts get-caller-identity

                    echo "===== AWS Region ====="
                    echo "${AWS_REGION}"
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    echo "===== Logging into ECR ====="

                    aws ecr get-login-password \
                        --region "${AWS_REGION}" | \
                    docker login \
                        --username AWS \
                        --password-stdin "${ECR_REGISTRY}"
                '''
            }
        }

        stage('Verify ECR Image') {
            steps {
                sh '''
                    echo "===== Checking ECR Image ====="

                    aws ecr describe-images \
                        --repository-name "${ECR_REPOSITORY}" \
                        --region "${AWS_REGION}" \
                        --query 'imageDetails[*].imageTags' \
                        --output json

                    echo "===== Pulling Application Image ====="

                    docker pull "${ECR_IMAGE}"
                '''
            }
        }

        stage('Configure EKS') {
            steps {
                sh '''
                    echo "===== Configuring kubeconfig ====="

                    aws eks update-kubeconfig \
                        --region "${AWS_REGION}" \
                        --name "${EKS_CLUSTER}"

                    echo "===== Current Kubernetes Context ====="

                    kubectl config current-context

                    echo "===== EKS Nodes ====="

                    kubectl get nodes -o wide
                '''
            }
        }

        stage('Deploy Namespace') {
            steps {
                sh '''
                    echo "===== Deploying Namespace ====="

                    kubectl apply -f namespace.yaml
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    echo "===== Deploying Application ====="

                    kubectl apply \
                        -f deployment.yaml \
                        -n "${NAMESPACE}"

                    kubectl apply \
                        -f service.yaml \
                        -n "${NAMESPACE}"
                '''
            }
        }

        stage('Wait for Deployment') {
            steps {
                sh '''
                    echo "===== Waiting for Deployment ====="

                    kubectl rollout status \
                        deployment/application-code \
                        -n "${NAMESPACE}" \
                        --timeout=180s
                '''
            }
        }

        stage('Verify Application') {
            steps {
                sh '''
                    echo "===== Pods ====="

                    kubectl get pods \
                        -n "${NAMESPACE}" \
                        -o wide

                    echo "===== Deployment ====="

                    kubectl get deployment \
                        -n "${NAMESPACE}"

                    echo "===== Service ====="

                    kubectl get svc \
                        -n "${NAMESPACE}"

                    echo "===== Service Details ====="

                    kubectl describe svc \
                        application-code-service \
                        -n "${NAMESPACE}"
                '''
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo '   K8S DEPLOYMENT SUCCESSFUL'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo '   K8S DEPLOYMENT FAILED'
            echo '======================================'
        }

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}
