pipeline {
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-cred')
        MOVIE_IMAGE = "newemira/movie-service"
        CAST_IMAGE = "newemira/cast-service"
        VERSION = "${BUILD_NUMBER}"
        ARGOCD_TOKEN = credentials('argocd-auth-token')
        ARGOCD_SERVER = "localhost:8082"
    }

    agent {
        docker {
            image 'docker:dind'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    stages {
        stage('Docker Build & Push'){
            steps {
                script {
                    sh '''
                    # Build des images
                    docker build -t $DOCKERHUB_CREDENTIALS/$MOVIE_IMAGE:$VERSION ${WORKSPACE}/movie-service
                    docker build -t $DOCKERHUB_CREDENTIALS/$CAST_IMAGE:$VERSION ${WORKSPACE}/cast-service
                    
                    # Login et push
                    docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PSW
                    docker push $DOCKERHUB_CREDENTIALS/$MOVIE_IMAGE:$VERSION
                    docker push $DOCKERHUB_CREDENTIALS/$CAST_IMAGE:$VERSION
                    '''
                }
            }
        }

        stage('Update Version & Sync Dev'){
            steps {
                script {
                    sh '''
                    argocd login ${ARGOCD_SERVER} --username admin --password ${ARGOCD_TOKEN} --insecure
                    argocd app set movie-service -p image.tag=${VERSION}
                    argocd app set cast-service -p image.tag=${VERSION}
                    argocd app sync movie-service cast-service
                    argocd app wait movie-service cast-service --health --timeout 300
                    '''
                }
            }
        }

        stage('Sync Staging'){
            steps {
                script {
                    sh '''
                    argocd app set movie-service-staging -p image.tag=${VERSION}
                    argocd app set cast-service-staging -p image.tag=${VERSION}
                    argocd app sync movie-service-staging cast-service-staging
                    argocd app wait movie-service-staging cast-service-staging --health --timeout 300
                    '''
                }
            }
        }

        stage('Sync Production'){
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production?', ok: 'Yes'
                }
                script {
                    sh '''
                    argocd app set movie-service-prod -p image.tag=${VERSION}
                    argocd app set cast-service-prod -p image.tag=${VERSION}
                    argocd app sync movie-service-prod cast-service-prod
                    argocd app wait movie-service-prod cast-service-prod --health --timeout 300
                    '''
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
