
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "nishanthhj02/productcatalogservice"
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
                sh 'go test ./...'
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
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

                        # Remove existing GitOps directory
                        if [ -d "gitops" ]; then
                            echo "Removing existing GitOps directory..."
                            rm -rf gitops
                        fi

                        # Clone GitOps repository
                        git clone \
                            "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/nishanthhj19/GitOps.git" \
                            gitops

                        # Correct Product Catalog path
                        cd gitops/productcatalogservice/

                        # Configure Git
                        git config user.email "jenkins@ci.com"
                        git config user.name "jenkins"

                        # Update image tag
                        echo "Updating Product Catalog image: ${IMAGE_NAME}"

                        sed -i \
                            "s|image: .*productcatalogservice.*|image: ${IMAGE_NAME}|g" \
                            deployment.yaml

                        # Check changes
                        if git diff --quiet; then
                            echo "No changes detected."
                            echo "Product Catalog image is already updated."
                            exit 0
                        fi

                        # Commit changes
                        git add deployment.yaml

                        git commit -m "Update productcatalogservice image to ${IMAGE_NAME}"

                        # Push changes
                        git push origin main

                        echo "Product Catalog GitOps update successful."
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
