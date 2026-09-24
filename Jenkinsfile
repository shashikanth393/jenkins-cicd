pipeline {
    agent any

    environment {
        // GCP Details (Auto-configured from your Cloud Shell prompt)
        GCP_PROJECT_ID         = 'my-project-2-508107'
        GKE_CLUSTER_NAME       = 'autopilot-cluster-1'
        GCP_REGION             = 'us-central1'             // Update if your cluster is in a different region (e.g. us-east1, asia-east1)
        ARTIFACT_REGISTRY_REPO = 'my-python-repo'         // Replace with your Artifact Registry repository name
        IMAGE_NAME             = 'hello-world-python'
        
        // Jenkins Credentials ID where your GCP Service Account JSON Key is stored
        GCP_CREDENTIALS_ID     = 'gcp-service-account-key'
        
        // Dynamic Registry Variables
        REGISTRY_HOST          = "${GCP_REGION}-docker.pkg.dev"
        FULL_IMAGE_NAME        = "${REGISTRY_HOST}/${GCP_PROJECT_ID}/${ARTIFACT_REGISTRY_REPO}/${IMAGE_NAME}"
        IMAGE_TAG              = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('Authenticate GCP & Docker') {
            steps {
                withCredentials([file(credentialsId: env.GCP_CREDENTIALS_ID, variable: 'GCP_KEY_FILE')]) {
                    sh '''
                        # Authenticate gcloud CLI
                        gcloud auth activate-service-account --key-file="${GCP_KEY_FILE}"
                        gcloud config set project ${GCP_PROJECT_ID}
                        
                        # Authenticate Docker for Artifact Registry
                        gcloud auth configure-docker ${REGISTRY_HOST} --quiet
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker Image: ${FULL_IMAGE_NAME}:${IMAGE_TAG}"
                    docker build -t ${FULL_IMAGE_NAME}:${IMAGE_TAG} -t ${FULL_IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Push Image to Artifact Registry') {
            steps {
                sh '''
                    echo "Pushing image to GCP Artifact Registry..."
                    docker push ${FULL_IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${FULL_IMAGE_NAME}:latest
                '''
            }
        }

        stage('Deploy to Autopilot GKE Cluster') {
            steps {
                withCredentials([file(credentialsId: env.GCP_CREDENTIALS_ID, variable: 'GCP_KEY_FILE')]) {
                    sh '''
                        # Connect to your GKE Autopilot Cluster
                        gcloud container clusters get-credentials ${GKE_CLUSTER_NAME} --region=${GCP_REGION} --project=${GCP_PROJECT_ID}
                        
                        # Replace image placeholder in k8s/deployment.yaml with the newly built image tag
                        sed -i "s|LOCATION-docker.pkg.dev/PROJECT_ID/REPOSITORY/IMAGE_NAME:TAG|${FULL_IMAGE_NAME}:${IMAGE_TAG}|g" k8s/deployment.yaml
                        
                        # Apply Kubernetes configuration
                        kubectl apply -f k8s/deployment.yaml
                        
                        # Verify deployment roll-out status
                        kubectl rollout status deployment/python-hello-world --timeout=180s
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Successfully deployed build #${BUILD_NUMBER} to ${GKE_CLUSTER_NAME}!"
        }
        failure {
            echo "Pipeline failed on build #${BUILD_NUMBER}. Please check logs."
        }
    }
}
