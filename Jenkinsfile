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

                sh '''
                    echo "Validating application files..."
                    test -f Dockerfile
                    test -f app/index.html
                    echo "Application build validation completed successfully."
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Running automated tests...'

                sh '''
                    test -f Dockerfile
                    test -f app/index.html
                    grep -q "Jenkins CI/CD Pipeline" app/index.html
                    grep -q "Version [0-9]" app/index.html
                '''

                echo 'All tests passed!'
            }
        }

        stage('Package') {
            steps {
                echo 'Packaging application...'

                sh '''
                    tar -czf ${APP_NAME}-${BUILD_NUMBER}.tar.gz \
                        app Dockerfile Jenkinsfile
                '''
            }
        }

        stage('Docker Build') {
            steps {
                echo 'Building Docker image with latest base image...'

                sh '''
                    docker build --pull --no-cache \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} .

                    docker tag \
                        ${IMAGE_NAME}:${IMAGE_TAG} \
                        ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Security Scan') {
    steps {
        echo 'Running Trivy security scan...'

        sh '''
            docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                aquasec/trivy:0.74.0 \
                image \
                --severity HIGH,CRITICAL \
                --exit-code 1 \
                ${IMAGE_NAME}:${IMAGE_TAG}
        '''

        echo 'Security scan passed: no HIGH or CRITICAL vulnerabilities found.'
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
                        echo "$DOCKER_PASSWORD" | \
                            docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying hardened application container...'

                sh '''
                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true

                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        --read-only \
                        --security-opt no-new-privileges:true \
                        --cap-drop ALL \
                        --tmpfs /var/cache/nginx \
                        --tmpfs /var/run \
                        --tmpfs /tmp \
                        -p ${HOST_PORT}:80 \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                '''

                echo 'Hardened application deployed successfully!'
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
                echo 'Recording stable release image...'

                sh '''
                    docker tag \
                        ${IMAGE_NAME}:${IMAGE_TAG} \
                        ${IMAGE_NAME}:stable

                    echo "Stable release image:"
                    docker images ${IMAGE_NAME}

                    echo "Release image tagged successfully."
                '''
            }
        }
    }

    post {

        success {
            echo '========================================'
            echo 'PRODUCTION READINESS PIPELINE PASSED'
            echo '========================================'
        }

        failure {
            echo '========================================'
            echo 'PIPELINE FAILED'
            echo 'Check the console output.'
            echo '========================================'
        }

        always {
            echo 'Production readiness pipeline execution completed.'
        }
    }
}



