pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "ansaarnaqvi12/straming-website" // Matches your successful login account
        DOCKER_HUB_CREDS = 'docker'
        KUBECONFIG_CREDS = 'k8s'
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
                    // Bypass the Kubernetes CLI plugin validator by writing kubeconfig manually
                    withCredentials([file(credentialsId: KUBECONFIG_CREDS, variable: 'KUBECONFIG_FILE')]) {
                        // Update the image in the deployment manifest
                        sh "sed -i 's|image: .*|image: ${DOCKER_IMAGE}:${env.BUILD_ID}|g' Kubernetes/deployment.yml"
                        sh "KUBECONFIG=${KUBECONFIG_FILE} kubectl apply -f Kubernetes/deployment.yml"
                        sh "KUBECONFIG=${KUBECONFIG_FILE} kubectl apply -f Kubernetes/service.yml"
                    }
                }
            }
        }

        stage('Prometheus/Grafana Stage') {
            steps {
                echo 'Verifying Deployment Status for Monitoring...'
                script {
                    withCredentials([file(credentialsId: KUBECONFIG_CREDS, variable: 'KUBECONFIG_FILE')]) {
                        sh "KUBECONFIG=${KUBECONFIG_FILE} kubectl get pods"
                        sh "KUBECONFIG=${KUBECONFIG_FILE} kubectl get svc"
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
