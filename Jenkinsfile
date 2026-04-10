pipeline {
    agent any

    environment {
        IMAGE_NAME = 'honeyshah062/modernchair'
        IMAGE_TAG = 'latest'
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials-id'
        K8S_NAMESPACE = 'app'
    }

    stages {

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $IMAGE_NAME:$IMAGE_TAG .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS_ID}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Load Image into Minikube') {
            steps {
                sh '''
                    minikube image load $IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    sh '''
                        kubectl apply -f my-nginx-app.yaml --validate=false
                        sleep 5
                    '''
                }
            }
        }

        stage('Print Ingress IP') {
            steps {
                sh '''
                    echo "Pods:"
                    kubectl get pods -n $K8S_NAMESPACE

                    echo "Services:"
                    kubectl get svc -n $K8S_NAMESPACE

                    echo "Ingress:"
                    kubectl get ingress -n $K8S_NAMESPACE -o wide

                    echo "Minikube IP:"
                    minikube ip
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check console output.'
        }
    }
}