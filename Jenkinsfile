pipeline {
    agent any

    stages {
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh 'docker build -t newemira/app:${BUILD_NUMBER} .'
            }
        }
        
        stage('Scan Docker Image with Trivy') {
            steps {
                echo 'Scanning Docker image with Trivy...'
                // Téléchargement de Trivy pour Jenkins si non installé
                sh '''
                if ! [ -x "$(command -v trivy)" ]; then
                    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
                fi
                '''
                // Scan de l'image Docker
                sh 'trivy image --exit-code 1 --severity CRITICAL votrecompte/app:${BUILD_NUMBER}'
            }
        }
        
        stage('Push Docker Image') {
            steps {
                echo 'Pushing Docker image to DockerHub...'
                sh 'docker push newemira/app:${BUILD_NUMBER}'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Updating Helm values and deploying...'
                sh '''
                sed -i 's/tag:.*/tag: ${BUILD_NUMBER}/' helm/app/values.yaml
                helm upgrade --install my-app helm/app -f helm/app/values.yaml --namespace dev
                '''
            }
        }
    }
}
