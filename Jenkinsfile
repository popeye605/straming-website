pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "popeye605/straming-website" // Changed to match your repo name/username style
        DOCKER_HUB_CREDS = 'docker'
        KUBECONFIG_CREDS = 'k8s-creds'
        TMDB_API_KEY = credentials('TMDB_API_KEY') // Add this to Jenkins Credentials as Secret Text
    }

    stages {
        stage('Code Fetch Stage') {
            steps {
                // Ensure the 'GitHub' plugin is installed and Webhook is configured
                git branch: 'main', 
                    url: 'https://github.com/popeye605/straming-website.git'
            }
        }

        stage('Docker Image Creation Stage') {
            steps {
                script {
                    // Requires 'Docker Pipeline' plugin
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_HUB_CREDS}") {
                        def appImage = docker.build("${DOCKER_IMAGE}:${env.BUILD_ID}", "--build-arg TMDB_V3_API_KEY=${TMDB_API_KEY} .")
                        appImage.push()
                        appImage.push('latest')
                    }
                }
            }
        }

        stage('Kubernetes Deployment Stage') {
            steps {
                script {
                    // Requires 'Kubernetes CLI' plugin
                    withKubeConfig([credentialsId: "${KUBECONFIG_CREDS}"]) {
                        // Update the image in the deployment manifest
                        sh "sed -i 's|image: .*|image: ${DOCKER_IMAGE}:${env.BUILD_ID}|g' Kubernetes/deployment.yml"
                        sh 'kubectl apply -f Kubernetes/deployment.yml'
                        sh 'kubectl apply -f Kubernetes/service.yml'
                    }
                }
            }
        }

        stage('Prometheus/Grafana Stage') {
            steps {
                echo 'Verifying Deployment Status for Monitoring...'
                script {
                    withKubeConfig([credentialsId: "${KUBECONFIG_CREDS}"]) {
                        sh 'kubectl get pods'
                        sh 'kubectl get svc'
                    }
                }
                echo 'Monitoring metrics should now be reflecting in Grafana dashboard.'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs.'
        }
    }
}
