pipeline {
    agent any

    environment {
        ACR_NAME = 'devopspracticeacr12345'
        AKS_RESOURCE_GROUP = 'devops-practice'
        AKS_CLUSTER_NAME = 'devops-practice-aks'
        NAMESPACE = 'robot-shop'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out robo-shop repository...'

                git branch: 'master',
                    url: 'https://github.com/saumitra-rajput/robo-shop.git'
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                    set -e

                    echo "===== USER ====="
                    whoami

                    echo "===== WORKSPACE ====="
                    pwd

                    echo "===== GIT ====="
                    git --version

                    echo "===== DOCKER ====="
                    docker --version

                    echo "===== AZURE CLI ====="
                    az version

                    echo "===== KUBECTL ====="
                    kubectl version --client

                    echo "===== HELM ====="
                    helm version
                '''
            }
        }

        stage('Verify Azure') {
            steps {
                sh '''
                    set -e

                    echo "===== AZURE ACCOUNT ====="
                    az account show -o table
                '''
            }
        }

        stage('Verify ACR') {
            steps {
                sh '''
                    set -e

                    echo "===== ACR ====="
                    az acr show \
                        --name "$ACR_NAME" \
                        --query "{Name:name,LoginServer:loginServer,Status:provisioningState}" \
                        -o table
                '''
            }
        }

        stage('Verify AKS') {
            steps {
                sh '''
                    set -e

                    echo "===== AKS CREDENTIALS ====="

                    az aks get-credentials \
                        --resource-group "$AKS_RESOURCE_GROUP" \
                        --name "$AKS_CLUSTER_NAME" \
                        --overwrite-existing

                    echo "===== AKS NODES ====="
                    kubectl get nodes

                    echo "===== ROBOT SHOP ====="
                    kubectl get pods -n "$NAMESPACE"
                '''
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo 'ROBOT SHOP PIPELINE PASSED'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'ROBOT SHOP PIPELINE FAILED'
            echo '======================================'
        }
    }
}
