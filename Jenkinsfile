

pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "nishanthhj02/productcatalogservice"
        GITOPS_REPO = "https://github.com/nishanthhj19/GitOps.git"
        GITOPS_BRANCH = "main"
    }

    stages {

        // 1. Git Checkout
        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/nishanthhj19/productcatalogservice.git',
                    branch: 'main'

                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=7 HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_NAME = "${env.DOCKER_IMAGE}:${env.IMAGE_TAG}"

                    echo "Building image: ${env.IMAGE_NAME}"
                }
            }
        }

        // 2. Run Unit Tests
        stage('Run Unit Tests') {
            steps {
                sh '''
                    set -e

                    echo "Running Go unit tests..."

                    go test ./...

                    echo "Unit tests completed successfully."
                '''
            }
        }

        // 3. Docker Build
        stage('Docker Build') {
            steps {
                sh '''
                    set -e

                    echo "Building Docker image: $IMAGE_NAME"

                    docker build -t "$IMAGE_NAME" .

                    echo "Docker build completed successfully."
                '''
            }
        }

        // 4. Login to Docker Hub
        stage('Login to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-creds',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin
                    '''
                }
            }
        }

        // 5. Push Docker Image
        stage('Push to Docker Hub') {
            steps {
                sh '''
                    set -e

                    echo "Pushing Docker image: $IMAGE_NAME"

                    docker push "$IMAGE_NAME"

                    echo "Docker image pushed successfully."
                '''
            }
        }

        // 6. Update GitOps Deployment
        stage('Update GitOps Deployment') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-creds',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -e

                        echo "Cloning GitOps repository..."

                        rm -rf gitops

                        git clone \
                            --branch "$GITOPS_BRANCH" \
                            "$GITOPS_REPO" \
                            gitops

                        cd gitops

                        # Configure Git
                        git config user.name "$GIT_USERNAME"
                        git config user.email "$GIT_USERNAME@users.noreply.github.com"

                        # Configure GitHub authentication
                        export GIT_TERMINAL_PROMPT=0
                        export GIT_ASKPASS="$PWD/git-askpass.sh"

                        cat > "$GIT_ASKPASS" <<'EOF'
#!/bin/sh
case "$1" in
    *Username*) echo "$GIT_USERNAME" ;;
    *Password*) echo "$GIT_PASSWORD" ;;
    *) exit 1 ;;
esac
EOF

                        chmod 700 "$GIT_ASKPASS"

                        # Cleanup authentication script on exit
                        trap 'rm -f "$GIT_ASKPASS"' EXIT

                        # Navigate to deployment directory
                        cd base/productcatalogservice

                        # Verify deployment file
                        if [ ! -f deployment.yaml ]; then
                            echo "ERROR: deployment.yaml not found"
                            exit 1
                        fi

                        echo "Updating Docker image..."

                        # Verify existing image line
                        if ! grep -Eq '^[[:space:]]*image:[[:space:]]*[^[:space:]]*productcatalogservice' deployment.yaml; then
                            echo "ERROR: Product Catalog image not found in deployment.yaml"
                            exit 1
                        fi

                        # Update Docker image
                        sed -i \
                            "s|\\(^[[:space:]]*image:[[:space:]]*\\).*productcatalogservice[^[:space:]]*|\\1${IMAGE_NAME}|" \
                            deployment.yaml

                        # Verify updated image
                        if ! grep -Fq "$IMAGE_NAME" deployment.yaml; then
                            echo "ERROR: Image update failed"
                            exit 1
                        fi

                        echo "Updated deployment image:"
                        grep -F "$IMAGE_NAME" deployment.yaml

                        # Return to GitOps repository root
                        cd ../..

                        # Commit deployment change
                        git add base/productcatalogservice/deployment.yaml

                        if git diff --cached --quiet; then
                            echo "No GitOps changes detected."
                        else
                            git commit -m "Update productcatalogservice image to ${IMAGE_TAG}"

                            # Push changes to GitHub
                            git push origin "HEAD:${GITOPS_BRANCH}"
                        fi

                        echo "GitOps deployment updated successfully."
                    '''
                }
            }
        }
    }

    // 7. Post Actions
    post {

        always {
            script {
                if (env.IMAGE_NAME) {
                    sh '''
                        docker rmi "$IMAGE_NAME" || true
                    '''
                }
            }

            sh '''
                docker logout || true
            '''
        }

        success {
            echo "Pipeline completed successfully!"
            echo "Docker image: ${env.IMAGE_NAME}"
            echo "GitOps deployment updated."
        }

        failure {
            echo "Pipeline failed. Check the logs above."
        }
    }
}
