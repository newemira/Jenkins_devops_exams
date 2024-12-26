pipeline {
    agent any

    environment {
        MOVIE_IMAGE = "movie-service"
        CAST_IMAGE = "cast-service"
        DOCKER_ID = "newemira"
        DOCKER_TAG =  "latest"
        ARGOCD_TOKEN = credentials('argocd-auth-token')
        ARGOCD_SERVER = "localhost:8082"
    }

    stages {
        stage('Docker Build Images'){
            parallel {
                stage('Build Movie Service') {
                    steps {
                        dir('movie-service') {
                            sh "docker build -t ${MOVIE_IMAGE}:$DOCKER_TAG ."
                        }
                    }
                }
                stage('Build Cast Service') {
                    steps {
                        dir('cast-service') {
                            sh "docker build -t ${CAST_IMAGE}:$DOCKER_TAG ."
                        }
                    }
                }
            }
        }

        stage('Docker run'){
            steps {
                script {
                        sh """
                        docker run -d -p 80:80 --name movie-service \
                           -e DATABASE_URI="postgresql://cast_db_username:cast_db_password@cast-db-service/cast_db_dev" \
                           ${DOCKER_ID}/${MOVIE_IMAGE}:${DOCKER_TAG}
                        docker run -d -p 81:81 --name cast-service ${DOCKER_ID}/${CAST_IMAGE}:$DOCKER_TAG
                        sleep 10
                        """
                    }
                }
            }
        

        stage('Test Acceptance'){
            steps {
                script {
                    sh """
                    curl localhost:80  # Vérification du movie-service
                    curl localhost:81  # Vérification du cast-service
                    """
                }
            }
        }    

        stage('Docker Push'){
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }

            steps {

                script {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker push ${DOCKER_ID}/${MOVIE_IMAGE}:$DOCKER_TAG
                docker push ${DOCKER_ID}/${CAST_IMAGE}:$DOCKER_TAG
                '''
                }
            }

        }

        stage('Update Version & Sync Dev'){
            steps {
                script {
                    sh '''
                    argocd login ${ARGOCD_SERVER} --username admin --password ${ARGOCD_TOKEN} --insecure
                    argocd app set movie-service -p image.tag=$DOCKER_TAG
                    argocd app set cast-service -p image.tag=$DOCKER_TAG
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
                    argocd app set movie-service-staging -p image.tag=$DOCKER_TAG
                    argocd app set cast-service-staging -p image.tag=$DOCKER_TAG
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
                    argocd app set movie-service-prod -p image.tag=$DOCKER_TAG
                    argocd app set cast-service-prod -p image.tag=$DOCKER_TAG
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
