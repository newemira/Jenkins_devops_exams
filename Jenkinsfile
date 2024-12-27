pipeline {
    agent any

    environment {
        MOVIE_IMAGE = "movie-service"
        CAST_IMAGE = "cast-service"
        DOCKER_ID = "newemira"
        DOCKER_TAG =  "latest"
        ARGOCD_TOKEN = credentials('ARGOCD_TOKEN')
        ARGOCD_SERVER = "127.0.0.1:8082"
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
                        docker run -d -p 80:80 --name movie-service ${DOCKER_ID}/${MOVIE_IMAGE}:$DOCKER_TAG
                        docker run -d -p 81:81 --name cast-service ${DOCKER_ID}/${CAST_IMAGE}:$DOCKER_TAG
                        sleep 10
                        """
                    }
                }
            }
        

        stage('Test Acceptance'){
            steps {
                script {
                    sh '''
                    docker ps
                    '''
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

    stage('Deploiement en dev'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp charts/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app fastapi --values=values.yml --namespace dev
                '''
                }
            }

        }
stage('Deploiement en staging'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp charts/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app fastapi --values=values.yml --namespace staging
                '''
                }
            }

        }
  stage('Deploiement en prod'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
            // Create an Approval Button with a timeout of 15minutes.
            // this require a manuel validation in order to deploy on production environment
                    timeout(time: 15, unit: "MINUTES") {
                        input message: 'Do you want to deploy in production ?', ok: 'Yes'
                    }

                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app fastapi --values=values.yml --namespace prod
                '''
                }
            }

        }

}
}
    
