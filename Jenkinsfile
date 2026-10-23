


pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "nishanthhj02/productcatalogservice"
        GITOPS_REPO = "https://github.com/nishanthhj19/GitOps.git"
        GITOPS_BRANCH = "main"
    }

    stages {

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

        stage('Run Unit Tests') {
            steps {
                sh '''
                    set -e
                    go test ./...
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -e
                    docker build -t "$IMAGE_NAME" .
                '''
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        set +x

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin
                    '''
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh '''
                    set -e
                    docker push "$IMAGE_NAME"
                '''
            }
        }

        stage('Update GitOps Deployment') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds',
                    usernameVariable: 'GIT_USERNAME',
                    passwordVariable: 'GIT_PASSWORD'
                )]) {

                    sh '''
                        set -e

                        # Clone GitOps repository
                        rm -rf gitops

                        git clone \
                            --branch "$GITOPS_BRANCH" \
                            "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/nishanthhj19/GitOps.git" \
                            gitops

                        # Navigate to correct deployment directory
                        cd gitops/base/productcatalogservice

                        # Verify deployment file exists
                        test -f deployment.yaml

                        # Configure Git
                        git config user.email "jenkins@ci.com"
                        git config user.name "jenkins"

                        # Update image
                        echo "Updating Product Catalog image:"
                        echo "$IMAGE_NAME"

                        sed -i \
                            "s|image: .*productcatalogservice.*|image: ${IMAGE_NAME}|g" \
                            deployment.yaml

                        # Verify updated image
                        grep -F "image: ${IMAGE_NAME}" deployment.yaml

                        # Stage changes
                        git add deployment.yaml

                        # Check changes
                        if git diff --cached --quiet; then
                            echo "No changes detected."
                            echo "Deployment image is already updated."
                        else
                            # Commit changes
                            git commit \
                                -m "Update Product Catalog image to ${IMAGE_NAME}"

                            # Push to GitOps main
                            git push origin "$GITOPS_BRANCH"

                            echo "GitOps deployment updated successfully."
                        fi
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                if (env.IMAGE_NAME) {
                    sh '''
                        docker rmi "$IMAGE_NAME" || true
                    '''
                }
            }

            sh 'docker logout || true'
        }

        success {
            echo "Build and push successful: ${env.IMAGE_NAME}"
        }

        failure {
            echo "Pipeline failed. Check the logs above."
        }
    }
}
