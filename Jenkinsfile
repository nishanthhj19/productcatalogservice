
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "nishanthhj02/productcatalogservice"
        GITOPS_REPO = "https://github.com/nishanthhj19/GitOps.git"
        GITOPS_BRANCH = "main"
    }

    stages {

        // ==============================
        // 1. Git Checkout
        // ==============================
        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/nishanthhj19/productcatalogservice.git',
                    branch: 'main'

                script {
                    env.GIT_COMMIT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_NAME = "${env.DOCKER_IMAGE}:${env.GIT_COMMIT}"

                    echo "Building image: ${env.IMAGE_NAME}"
                }
            }
        }

        // ==============================
        // 2. Run Unit Tests
        // ==============================
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

        // ==============================
        // 3. Docker Build
        // ==============================
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

        // ==============================
        // 4. Login to Docker Hub
        // ==============================
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

        // ==============================
        // 5. Push Docker Image
        // ==============================
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

        // ==============================
        // 6. Update GitOps Deployment
        // ==============================
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

                        // Configure Git
                        git config user.email "jenkins@ci.com"
                        git config user.name "jenkins"

                        // Configure temporary GitHub authentication
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

                        // Navigate to Product Catalog deployment
                        cd base/productcatalogservice

                        // Verify deployment file
                        if [ ! -f deployment.yaml ]; then
                            echo "ERROR: deployment.yaml not found"
                            exit 1
                        fi

                        // Update Docker image
                        echo "Updating Product Catalog image:"
                        echo "$IMAGE_NAME"

                        sed -i \
                            "s|image: .*productcatalogservice.*|image: ${IMAGE_NAME}|g" \
                            deployment.yaml

                        // Verify image update
                        grep -F "image: ${IMAGE_NAME}" deployment.yaml

                        // Stage changes
                        git add deployment.yaml

                        // Check changes
                        if git diff --cached --quiet; then
                            echo "No changes detected."
                            echo "Deployment image is already updated."
                        else

                            // Commit changes
                            git commit \
                                -m "Update Product Catalog image to ${IMAGE_NAME}"

                            // Push changes
                            git push origin "$GITOPS_BRANCH"

                            echo "GitOps deployment updated successfully."
                        fi
                    '''
                }
            }
        }
    }

    // ==============================
    // 7. Post Actions
    // ==============================
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
