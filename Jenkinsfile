pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              serviceAccountName: jenkins-sa
              containers:
              - name: kaniko
                image: gcr.io/kaniko-project/executor:debug
                command:
                - sleep
                args:
                - 9999999
              - name: cloud-sdk
                image: google/cloud-sdk:latest
                command:
                - sleep
                args:
                - 9999999
            '''
        }
    }

    environment {
        // GCP & GKE Configuration
        GCP_PROJECT_ID         = 'my-project-2-508107'
        GKE_CLUSTER_NAME       = 'autopilot-cluster-1'
        GCP_REGION             = 'us-central1'
        ARTIFACT_REGISTRY_REPO = 'my-python-repo'
        IMAGE_NAME             = 'hello-world-python'
        
        // Derived Image Path
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

        stage('Build & Push Image (Kaniko)') {
            steps {
                container('kaniko') {
                    sh '''
                        /kaniko/executor --context `pwd` \
                            --destination ${FULL_IMAGE_NAME}:${IMAGE_TAG} \
                            --destination ${FULL_IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy to Autopilot GKE Cluster') {
            steps {
                container('cloud-sdk') {
                    sh '''
                        # Connect to GKE Autopilot Cluster
                        gcloud container clusters get-credentials ${GKE_CLUSTER_NAME} --location=${GCP_REGION} --project=${GCP_PROJECT_ID}
                        
                        # Substitute image path in deployment manifest
                        sed -i "s|LOCATION-docker.pkg.dev/PROJECT_ID/REPOSITORY/IMAGE_NAME:TAG|${FULL_IMAGE_NAME}:${IMAGE_TAG}|g" k8s/deployment.yaml
                        
                        # Apply Kubernetes configuration
                        kubectl apply -f k8s/deployment.yaml
                        
                        # Verify rollout status
                        kubectl rollout status deployment/python-hello-world --timeout=180s
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Successfully deployed build #${BUILD_NUMBER} to${GKE_CLUSTER_NAME}!"
        }
        failure {
            echo "Pipeline failed on build #${BUILD_NUMBER}. Please check logs."
        }
    }
}
