pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-cred')
        MOVIE_IMAGE = "emilebernard/movie-service"
        CAST_IMAGE = "emilebernard/cast-service"
        VERSION = "${BUILD_NUMBER}"
        GIT_CREDENTIALS = credentials('github-cred')
        ARGOCD_TOKEN = credentials('argocd-auth-token')  
        ARGOCD_SERVER = "argocd.devops-emi.cloudns.ch"  
    }

    stages {
        stage('Build Docker Images') {
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

        stage('Scan Images with Trivy') {
            parallel {
                stage('Scan Movie Service') {
                    steps {
                        sh """
                            if ! [ -x "\$(command -v trivy)" ]; then
                                curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
                            fi
                            trivy image --exit-code 1 --severity HIGH,CRITICAL ${MOVIE_IMAGE}:${VERSION}
                        """
                    }
                }
                stage('Scan Cast Service') {
                    steps {
                        sh """
                            trivy image --exit-code 1 --severity HIGH,CRITICAL ${CAST_IMAGE}:${VERSION}
                        """
                    }
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                sh """
                    echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin
                    docker push ${MOVIE_IMAGE}:${VERSION}
                    docker push ${CAST_IMAGE}:${VERSION}
                """
            }
        }

        stage('Update Dev Manifests') {
            steps {
                sh """
                    # Cloner le repo des manifests
                    git clone https://github.com/votre-username/k8s-manifests.git
                    cd k8s-manifests
                    
                    # Mettre à jour les tags des images dans values-dev.yaml
                    yq e '.movieService.image.tag = "${VERSION}"' -i environments/dev/values.yaml
                    yq e '.castService.image.tag = "${VERSION}"' -i environments/dev/values.yaml
                    
                    # Commit et push des changements
                    git config user.email "jenkins@example.com"
                    git config user.name "Jenkins"
                    git add environments/dev/values.yaml
                    git commit -m "Update dev images to version ${VERSION}"
                    git push https://${GIT_CREDENTIALS_USR}:${GIT_CREDENTIALS_PSW}@github.com/votre-username/k8s-manifests.git
                """
                
                sh """
                    argocd login ${ARGOCD_SERVER} --username admin --password ${ARGOCD_TOKEN} --insecure
                    argocd app sync movie-cast-dev
                    argocd app wait movie-cast-dev --health --timeout 300
                """
            }
        }

        stage('Test in Dev') {
            steps {
                sh """
                    # Attendre que les pods soient prêts
                    kubectl wait --for=condition=ready pod -l app=movie-service -n dev --timeout=180s
                    kubectl wait --for=condition=ready pod -l app=cast-service -n dev --timeout=180s
                    
                    # Test des endpoints
                    curl -f http://app.devops-emi.cloudns.ch/api/v1/movies/docs
                    curl -f http://app.devops-emi.cloudns.ch/api/v1/casts/docs
                """
            }
        }

        stage('Deploy to QA') {
            when {
                expression { currentBuild.resultIsBetterOrEqualTo('SUCCESS') }
            }
            steps {
                sh """
                    cd k8s-manifests
                    yq e '.movieService.image.tag = "${VERSION}"' -i environments/qa/values.yaml
                    yq e '.castService.image.tag = "${VERSION}"' -i environments/qa/values.yaml
                    git add environments/qa/values.yaml
                    git commit -m "Update qa images to version ${VERSION}"
                    git push https://${GIT_CREDENTIALS_USR}:${GIT_CREDENTIALS_PSW}@github.com/votre-username/k8s-manifests.git
                    
                    # Sync QA environment
                    argocd app sync movie-cast-qa
                    argocd app wait movie-cast-qa --health --timeout 300
                """
            }
        }

        stage('Test in QA') {
            steps {
                sh """
                    kubectl wait --for=condition=ready pod -l app=movie-service -n qa --timeout=180s
                    kubectl wait --for=condition=ready pod -l app=cast-service -n qa --timeout=180s
                    
                    curl -f http://qa.app.devops-emi.cloudns.ch/api/v1/movies/docs
                    curl -f http://qa.app.devops-emi.cloudns.ch/api/v1/casts/docs
                """
            }
        }

        stage('Deploy to Staging') {
            when {
                expression { currentBuild.resultIsBetterOrEqualTo('SUCCESS') }
            }
            steps {
                sh """
                    cd k8s-manifests
                    yq e '.movieService.image.tag = "${VERSION}"' -i environments/staging/values.yaml
                    yq e '.castService.image.tag = "${VERSION}"' -i environments/staging/values.yaml
                    git add environments/staging/values.yaml
                    git commit -m "Update staging images to version ${VERSION}"
                    git push https://${GIT_CREDENTIALS_USR}:${GIT_CREDENTIALS_PSW}@github.com/votre-username/k8s-manifests.git
                    
                    # Sync Staging environment
                    argocd app sync movie-cast-staging
                    argocd app wait movie-cast-staging --health --timeout 300
                """
            }
        }

        stage('Test in Staging') {
            steps {
                sh """
                    kubectl wait --for=condition=ready pod -l app=movie-service -n staging --timeout=180s
                    kubectl wait --for=condition=ready pod -l app=cast-service -n staging --timeout=180s
                    
                    curl -f http://staging.app.devops-emi.cloudns.ch/api/v1/movies/docs
                    curl -f http://staging.app.devops-emi.cloudns.ch/api/v1/casts/docs
                """
            }
        }

        stage('Deploy to Production') {
            when {
                expression { currentBuild.resultIsBetterOrEqualTo('SUCCESS') }
            }
            steps {
                input message: 'Deploy to production?'
                sh """
                    cd k8s-manifests
                    yq e '.movieService.image.tag = "${VERSION}"' -i environments/production/values.yaml
                    yq e '.castService.image.tag = "${VERSION}"' -i environments/production/values.yaml
                    git add environments/production/values.yaml
                    git commit -m "Update production images to version ${VERSION}"
                    git push https://${GIT_CREDENTIALS_USR}:${GIT_CREDENTIALS_PSW}@github.com/votre-username/k8s-manifests.git
                    
                    # Sync Production environment
                    argocd app sync movie-cast-production
                    argocd app wait movie-cast-production --health --timeout 300
                """
            }
        }

        stage('Deploy to Dev') {
            steps {
                sh """
                    helm upgrade --install movie-cast charts/ --namespace dev
                    helm test movie-cast --namespace dev
                """
            }
        }

        stage('Deploy to QA') {
            steps {
                sh """
                    helm upgrade --install movie-cast charts/ --namespace qa
                    helm test movie-cast --namespace qa
                """
            }
        }

        stage('Deploy to Staging') {
            steps {
                sh """
                    helm upgrade --install movie-cast charts/ --namespace staging
                    helm test movie-cast --namespace staging
                """
            }
        }

        stage('Deploy to Production') {
            steps {
                sh """
                    helm upgrade --install movie-cast charts/ --namespace production
                    helm test movie-cast --namespace production
                """
            }
        }
    }

    post {
        always {
            sh """
                docker logout
                rm -rf k8s-manifests
                argocd logout ${ARGOCD_SERVER}
            """
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
