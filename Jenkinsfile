pipeline {
    agent any

    // allow overriding IMAGE_TAG via build parameters or env
    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'v2', description: 'Docker image tag to build and push')
    }

    environment {
        DOCKER_HUB_REPO = "rachanabaldania/studybuddyai"
        DOCKER_HUB_CREDENTIALS_ID = "dockerhub-token"
        GIT_CREDENTIALS_ID = "github-token"
    }

    options {
        // don't auto-checkout; we'll control checkout explicitly
        skipDefaultCheckout(true)
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                // Use the git step (simpler and supported) and explicit credentials
                git branch: 'main',
                    url: 'https://github.com/Rachana-Baldania/Study_Buddy_AI.git',
                    credentialsId: "${GIT_CREDENTIALS_ID}"
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    echo "Building Docker image ${DOCKER_HUB_REPO}:${params.IMAGE_TAG} ..."
                    // build and push in the same stage so the image object is available
                    def image = docker.build("${DOCKER_HUB_REPO}:${params.IMAGE_TAG}")

                    echo "Pushing Docker image to Docker Hub..."
                    // Use Docker Hub registry URL commonly used by docker.withRegistry
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_HUB_CREDENTIALS_ID}") {
                        image.push("${params.IMAGE_TAG}")
                        // optionally push a 'latest' tag:
                        // image.push('latest')
                    }
                }
            }
        }

        // Uncomment and adapt these stages if you want the pipeline to update manifests in-repo
        /*
        stage('Update Deployment YAML with New Tag') {
            steps {
                script {
                    sh """
                    sed -i 's|image: .*|image: ${DOCKER_HUB_REPO}:${params.IMAGE_TAG}|' manifests/deployment.yaml
                    """
                }
            }
        }

        stage('Commit Updated YAML') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: GIT_CREDENTIALS_ID, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        sh '''
                        git config user.name "ci-bot"
                        git config user.email "ci-bot@example.com"
                        git add manifests/deployment.yaml
                        git commit -m "Update image tag to ${IMAGE_TAG}" || echo "No changes to commit"
                        git push https://${GIT_USER}:${GIT_PASS}@github.com/Rachana-Baldania/Study_Buddy_AI.git HEAD:main
                        '''
                    }
                }
            }
        }
        */

        stage('Install kubectl & argocd CLI') {
            steps {
                sh '''
                echo 'Installing kubectl & argocd CLI...'
                curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                chmod +x kubectl
                mv kubectl /usr/local/bin/kubectl

                curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
                chmod +x /usr/local/bin/argocd

                # show versions
                kubectl version --client || true
                argocd version --client || true
                '''
            }
        }

        stage('Sync ArgoCD Application') {
            steps {
                script {
                    // Expecting a file-type credential that contains kubeconfig; adjust credentialsId as needed.
                    // If you have a different credential type, change this block to match (e.g., username/password or secret text).
                    withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                        export KUBECONFIG=${KUBECONFIG_FILE}
                        echo "KUBECONFIG set to file credential"

                        # Login to argocd (adjust argocd server host/port if needed)
                        argocd login 35.188.77.25:31704 --username admin --password $(kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d) --insecure

                        # sync the app named 'study'
                        argocd app sync study
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning workspace...'
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully. Image: ${DOCKER_HUB_REPO}:${params.IMAGE_TAG}"
        }
        failure {
            echo "Pipeline failed."
        }
    }
}
