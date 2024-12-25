pipeline {
    environment {
        DOCKER_ID = credentials('dockerhub-cred').username
        MOVIE_IMAGE = "newemira/movie-service"
        CAST_IMAGE = "newemira/cast-service"
        VERSION = "${BUILD_NUMBER}"
        ARGOCD_TOKEN = credentials('argocd-auth-token')
        ARGOCD_SERVER = "localhost:8082"
    }

    agent any

    stages {
        stage('Docker Build Images') {
            parallel {
                stage('Build Movie Service') {
                    steps {
                        dir('movie-service') {
                            sh "docker build -t ${MOVIE_IMAGE}:${VERSION} ."
                        }
                    }
                }
                stage('Build Cast Service') {
                    steps {
                        dir('cast-service') {
                            sh "docker build -t ${CAST_IMAGE}:${VERSION} ."
                        }
                    }
                }
            }
        }

        stage('Docker run') {
            steps {
                script {
                    sh """
                    docker run -d -p 80:80 --name jenkins ${DOCKER_ID}/${MOVIE_IMAGE}:${VERSION}
                    docker run -d -p 8081:80 --name cast-service ${DOCKER_ID}/${CAST_IMAGE}:${VERSION}
                    sleep 10
                    """
                }
            }
        }

        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                    curl localhost:8080  # Teste le conteneur movie-service
                    curl localhost:8081  # Teste le conteneur cast-service
                    '''
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials('docker_hub_pass') // Assurez-vous que ce secret existe
            }

            steps {
                script {
                    sh """
                    docker login -u $DOCKER_ID -p $DOCKER_PASS
                    docker push $DOCKER_ID/$MOVIE_IMAGE:$VERSION
                    docker push $DOCKER_ID/$CAST_IMAGE:$VERSION
                    """
                }
            }
        }

        stage('Update Version & Sync Dev') {
            steps {
                script {
                    sh """
                    argocd login ${ARGOCD_SERVER} --username admin --password ${ARGOCD_TOKEN} --insecure
                    argocd app set movie-service -p image.tag=${VERSION}
                    argocd app set cast-service -p image.tag=${VERSION}
                    argocd app sync movie-service cast-service
                    argocd app wait movie-service cast-service --health --timeout 300
                    """
                }
            }
        }

        stage('Sync Staging') {
            steps {
                script {
                    sh """
                    argocd app set movie-service-staging -p image.tag=${VERSION}
                    argocd app set cast-service-staging -p image.tag=${VERSION}
                    argocd app sync movie-service-staging cast-service-staging
                    argocd app wait movie-service-staging cast-service-staging --health --timeout 300
                    """
                }
            }
        }

        stage('Sync Production') {
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production?', ok: 'Yes'
                }
                script {
                    sh """
                    argocd app set movie-service-prod -p image.tag=${VERSION}
                    argocd app set cast-service-prod -p image.tag=${VERSION}
                    argocd app sync movie-service-prod cast-service-prod
                    argocd app wait movie-service-prod cast-service-prod --health --timeout 300
                    """
                }
            }
        }
    }

    post {
        always {
            sh '''
            docker logout
            argocd logout ${ARGOCD_SERVER}
            '''
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
