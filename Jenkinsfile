pipeline {
    agent any

    environment {
        ACR_NAME           = 'devopspracticeacr12345'
        ACR_LOGIN_SERVER   = "${ACR_NAME}.azurecr.io"
        AKS_RESOURCE_GROUP = 'devops-practice'
        AKS_CLUSTER_NAME   = 'devops-practice-aks'
        NAMESPACE          = 'robot-shop'

        // TODO: fill in with `az account show --query tenantId -o tsv`
        AZURE_TENANT_ID    = '2df8cf0c-d6d9-45a2-9ba5-9cbd5e071b72'

        // All 10 custom-built services share one image repo/version pair,
        // matching AKS/helm/values.yaml's image.repo / image.version.
        IMAGE_REPO = "${ACR_LOGIN_SERVER}/robotshop"
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
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

                    echo "===== AZURE CLI ====="
                    az version

                    echo "===== KUBECTL ====="
                    kubectl version --client

                    echo "===== HELM ====="
                    helm version
                '''
            }
        }

        // ---------------------------------------------------------------
        // Logs in once, here -- every later stage runs in this same
        // workspace/container, so the az CLI's cached token (~/.azure)
        // carries through the rest of the pipeline without re-logging in.
        // ---------------------------------------------------------------
        stage('Azure Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'acr-sp-credentials',
                    usernameVariable: 'AZ_CLIENT_ID',
                    passwordVariable: 'AZ_CLIENT_SECRET'
                )]) {
                    sh '''
                        set -e
                        # --allow-no-subscriptions: this SP is only granted narrow,
                        # resource-scoped roles (not Contributor), so it may not see
                        # any subscriptions via the subscription-list API. Without
                        # this flag, az login treats that as a hard failure even
                        # though authentication itself succeeded.
                        az login --service-principal \
                            -u "$AZ_CLIENT_ID" -p "$AZ_CLIENT_SECRET" --tenant "$AZURE_TENANT_ID" \
                            --allow-no-subscriptions >/dev/null
                        echo "Logged in."
                    '''
                }
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

        // ---------------------------------------------------------------
        // Build & push every custom-built service via Kaniko (baked into
        // this Jenkins image). Kaniko builds+pushes directly using plain
        // Docker-registry basic auth against the ACR -- it only needs the
        // AcrPush data-plane role, never any ARM/subscription permission
        // like az acr build's scheduleRun action. No Docker daemon needed
        // either way, since AKS nodes run containerd.
        // ---------------------------------------------------------------
        stage('Build & Push Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'acr-sp-credentials',
                    usernameVariable: 'AZ_CLIENT_ID',
                    passwordVariable: 'AZ_CLIENT_SECRET'
                )]) {
                    sh '''
                        set -e

                        # Build a Docker registry auth file for Kaniko. Running as the
                        # non-root "jenkins" user, so this goes in the workspace, not
                        # the default /kaniko/.docker/ -- pointed at via DOCKER_CONFIG.
                        DOCKER_CONFIG_DIR="$WORKSPACE/.kaniko-docker"
                        mkdir -p "$DOCKER_CONFIG_DIR"
                        AUTH=$(printf '%s:%s' "$AZ_CLIENT_ID" "$AZ_CLIENT_SECRET" | base64 -w0)
                        cat > "$DOCKER_CONFIG_DIR/config.json" <<EOF
{
  "auths": {
    "${ACR_LOGIN_SERVER}": {
      "auth": "${AUTH}"
    }
  }
}
EOF
                        export DOCKER_CONFIG="$DOCKER_CONFIG_DIR"

                        # folder-name -> image-name (matches AKS/helm's rs-<service> convention;
                        # mongo/mysql are the two irregular ones)
                        SERVICES="web cart catalogue dispatch mongo mysql payment ratings shipping user"

                        for svc in $SERVICES; do
                            case "$svc" in
                                mongo) image_name="rs-mongodb" ;;
                                mysql) image_name="rs-mysql-db" ;;
                                *)     image_name="rs-$svc" ;;
                            esac

                            echo "==> Building & pushing ${IMAGE_REPO}/${image_name}:${IMAGE_TAG} (context: ./$svc)"
                            /kaniko/executor \
                                --context "dir://${WORKSPACE}/${svc}" \
                                --dockerfile "${WORKSPACE}/${svc}/Dockerfile" \
                                --destination "${IMAGE_REPO}/${image_name}:${IMAGE_TAG}"
                        done

                        rm -rf "$DOCKER_CONFIG_DIR"
                    '''
                }
            }
        }

        // ---------------------------------------------------------------
        // Deploy: point the Helm release at the freshly built tag.
        // ---------------------------------------------------------------
        stage('Deploy (Helm)') {
            steps {
                sh '''
                    set -e

                    az aks get-credentials \
                        --resource-group "$AKS_RESOURCE_GROUP" \
                        --name "$AKS_CLUSTER_NAME" \
                        --overwrite-existing

                    helm upgrade --install robot-shop ./AKS/helm \
                        --namespace "$NAMESPACE" --create-namespace \
                        --set image.repo="$IMAGE_REPO" \
                        --set image.version="$IMAGE_TAG" \
                        --wait --timeout 5m
                '''
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo "ROBOT SHOP PIPELINE PASSED -- deployed image tag: ${IMAGE_TAG}"
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'ROBOT SHOP PIPELINE FAILED'
            echo '======================================'
        }
    }
}
