pipeline {

    agent any

    environment {
        APP_NAME = "jenkins-docker-cicd"
        IMAGE_NAME = "gowthu04/jenkins-docker-cicd"
        IMAGE_TAG = "${BUILD_NUMBER}"
        CONTAINER_NAME = "jenkins-docker-cicd-app"
        HOST_PORT = "8081"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code from GitHub...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Building application...'
                sh 'echo Application build completed successfully'
            }
        }

        stage('Test') {
            steps {
                echo 'Running automated tests...'

                sh 'test -f Dockerfile'
                sh 'test -f app/index.html'
                sh 'grep -q "Jenkins CI/CD Pipeline" app/index.html'
                sh 'grep -q "Version [0-9]" app/index.html'

                echo 'All tests passed!'
            }
        }

        stage('Package') {
            steps {
                echo 'Packaging application...'

                sh 'tar -czf ${APP_NAME}-${BUILD_NUMBER}.tar.gz app Dockerfile Jenkinsfile'
            }
        }

        stage('Docker Build') {
            steps {
                echo 'Building Docker image...'

                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
                sh 'docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest'
            }
        }

        stage('Docker Push') {
            steps {
                echo 'Pushing Docker image to Docker Hub...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying application...'

                sh '''
                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true

                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${HOST_PORT}:80 \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                '''

                echo 'Application deployed successfully!'
            }
        }

        stage('Verify') {
            steps {
                echo 'Verifying deployed application...'

                sh '''
                    sleep 5

                    curl -f http://host.docker.internal:${HOST_PORT}

                    echo ""
                    echo "Application verification successful!"
                '''
            }
        }

        stage('Rolling Deployment') {
            steps {
                echo 'Performing rolling deployment...'

                sh '''
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:stable

                    echo "Current image:"
                    docker images ${IMAGE_NAME}

                    echo "Rolling deployment completed successfully!"
                '''
            }
        }
    }

    post {

        success {
            echo '========================================'
            echo 'CI/CD PIPELINE COMPLETED SUCCESSFULLY'
            echo '========================================'
        }

        failure {
            echo '========================================'
            echo 'CI/CD PIPELINE FAILED'
            echo 'Check the console output.'
            echo '========================================'
        }
    }
}

