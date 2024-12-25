pipeline {
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-cred') // Identifiants Docker Hub
        MOVIE_IMAGE = "newemira/movie-service"
        CAST_IMAGE = "newemira/cast-service"
        VERSION = "${BUILD_NUMBER}" // Utilisation du numéro de build comme version
        GIT_CREDENTIALS = credentials('github-cred') // Identifiants GitHub
        ARGOCD_TOKEN = credentials('argocd-auth-token')  // Token pour ArgoCD
        ARGOCD_SERVER = "https://localhost:8082/applications"  // URL de votre serveur ArgoCD
        KUBECONFIG = credentials("config") // Récupération du kubeconfig pour les déploiements Kubernetes
    }

    agent any // Jenkins choisira un agent disponible

    stages {
        stage('Docker Build'){
            steps {
                script {
                    sh '''
                    docker rm -f jenkins || true
                    docker build -t $DOCKERHUB_CREDENTIALS/$MOVIE_IMAGE:$VERSION .
                    docker build -t $DOCKERHUB_CREDENTIALS/$CAST_IMAGE:$VERSION .
                    sleep 6
                    '''
                }
            }
        }

        stage('Docker Run'){
            steps {
                script {
                    sh '''
                    docker run -d -p 80:80 --name jenkins $DOCKERHUB_CREDENTIALS/$MOVIE_IMAGE:$VERSION
                    docker run -d -p 81:81 --name jenkins-cast $DOCKERHUB_CREDENTIALS/$CAST_IMAGE:$VERSION
                    sleep 10
                    '''
                }
            }
        }

        stage('Test Acceptance'){
            steps {
                script {
                    sh '''
                    curl localhost:80
                    curl localhost:81
                    '''
                }
            }
        }

        stage('Docker Push'){
            steps {
                script {
                    sh '''
                    docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PSW
                    docker push $DOCKERHUB_CREDENTIALS/$MOVIE_IMAGE:$VERSION
                    docker push $DOCKERHUB_CREDENTIALS/$CAST_IMAGE:$VERSION
                    '''
                }
            }
        }

        stage('Deploiement en dev'){
            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    cp fastapi/values.yaml values.yml
                    sed -i "s+tag.*+tag: ${VERSION}+g" values.yml
                    helm upgrade --install app fastapi --values=values.yml --namespace dev
                    '''
                }
            }
        }

        stage('Synchroniser avec ArgoCD pour Dev'){
            steps {
                script {
                    sh '''
                    argocd login ${ARGOCD_SERVER} --username admin --password ${ARGOCD_TOKEN} --insecure
                    argocd app sync app-dev --namespace dev
                    argocd app wait app-dev --health --timeout 300
                    '''
                }
            }
        }

        stage('Deploiement en staging'){
            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    cp fastapi/values.yaml values.yml
                    sed -i "s+tag.*+tag: ${VERSION}+g" values.yml
                    helm upgrade --install app fastapi --values=values.yml --namespace staging
                    '''
                }
            }
        }

        stage('Synchroniser avec ArgoCD pour Staging'){
            steps {
                script {
                    sh '''
                    argocd login ${ARGOCD_SERVER} --username admin --password ${ARGOCD_TOKEN} --insecure
                    argocd app sync app-staging --namespace staging
                    argocd app wait app-staging --health --timeout 300
                    '''
                }
            }
        }

        stage('Deploiement en prod'){
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production?', ok: 'Yes'
                }
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    cp fastapi/values.yaml values.yml
                    sed -i "s+tag.*+tag: ${VERSION}+g" values.yml
                    helm upgrade --install app fastapi --values=values.yml --namespace prod
                    '''
                }
            }
        }

        stage('Synchroniser avec ArgoCD pour Production'){
            steps {
                script {
                    sh '''
                    argocd login ${ARGOCD_SERVER} --username admin --password ${ARGOCD_TOKEN} --insecure
                    argocd app sync app-prod --namespace prod
                    argocd app wait app-prod --health --timeout 300
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
            docker logout
            rm -rf .kube
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
