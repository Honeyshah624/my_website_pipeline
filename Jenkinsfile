// pipeline {
//     agent any

//     environment {
//         IMAGE_NAME = 'honeyshah062/modernchair'
//         IMAGE_TAG = 'latest'
//         DOCKER_CREDENTIALS_ID = 'dockerhub-credentials-id'
//         K8S_NAMESPACE = 'app'
//     }

//     stages {

//         stage('Build Docker Image') {
//             steps {
//                 sh '''
//                     docker build -t $IMAGE_NAME:$IMAGE_TAG .
//                 '''
//             }
//         }

//         stage('Push Docker Image') {
//             steps {
//                 withCredentials([usernamePassword(
//                     credentialsId: "${DOCKER_CREDENTIALS_ID}",
//                     usernameVariable: 'DOCKER_USER',
//                     passwordVariable: 'DOCKER_PASS'
//                 )]) {
//                     sh '''
//                         echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
//                         docker push $IMAGE_NAME:$IMAGE_TAG
//                     '''
//                 }
//             }
//         }

//         stage('Load Image into Minikube') {
//             steps {
//                 sh '''
//                     minikube image load $IMAGE_NAME:$IMAGE_TAG
//                 '''
//             }
//         }

//         stage('Deploy to Kubernetes') {
//             steps {
//                 timeout(time: 2, unit: 'MINUTES') {
//                     sh '''
//                         kubectl apply -f my-nginx-app.yaml --validate=false
//                         sleep 5
//                     '''
//                 }
//             }
//         }

//         stage('Print Ingress IP') {
//             steps {
//                 sh '''
//                     echo "Pods:"
//                     kubectl get pods -n $K8S_NAMESPACE

//                     echo "Services:"
//                     kubectl get svc -n $K8S_NAMESPACE

//                     echo "Ingress:"
//                     kubectl get ingress -n $K8S_NAMESPACE -o wide

//                     echo "Minikube IP:"
//                     minikube ip
//                 '''
//             }
//         }
//     }

//     post {
//         success {
//             echo 'Pipeline executed successfully.'
//         }
//         failure {
//             echo 'Pipeline failed. Check console output.'
//         }
//     }
// }

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
                script {
                    echo "Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                }
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
                script {
                    echo "Loading image into Minikube"
                }
                sh '''
                    minikube image load $IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Verify Minikube Status') {
            steps {
                sh '''
                    echo "Checking Minikube status..."
                    minikube status
                    
                    echo "Minikube IP:"
                    minikube ip
                    
                    echo "Checking kubectl connectivity..."
                    kubectl cluster-info
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    sh '''
                        # Ensure we're using the correct context
                        kubectl config use-context minikube
                        
                        # Create namespace if it doesn't exist
                        kubectl create namespace $K8S_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Verify namespace is created
                        kubectl get namespace $K8S_NAMESPACE
                        
                        # Apply Kubernetes manifests
                        kubectl apply -f my-nginx-app.yaml
                        
                        # Wait for deployment to be ready
                        echo "Waiting for deployment to be ready..."
                        kubectl wait --for=condition=available --timeout=90s deployment/my-nginx-app -n $K8S_NAMESPACE || true
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "================================"
                    echo "Deployment Status:"
                    echo "================================"
                    kubectl get deployments -n $K8S_NAMESPACE
                    
                    echo ""
                    echo "================================"
                    echo "Pods:"
                    echo "================================"
                    kubectl get pods -n $K8S_NAMESPACE -o wide
                    
                    echo ""
                    echo "================================"
                    echo "Services:"
                    echo "================================"
                    kubectl get svc -n $K8S_NAMESPACE
                    
                    echo ""
                    echo "================================"
                    echo "Ingress:"
                    echo "================================"
                    kubectl get ingress -n $K8S_NAMESPACE -o wide
                    
                    echo ""
                    echo "================================"
                    echo "Pod Logs (if available):"
                    echo "================================"
                    kubectl logs -n $K8S_NAMESPACE -l app=my-nginx-app --tail=50 || echo "No logs available yet"
                '''
            }
        }

        stage('Setup Ingress Access') {
            steps {
                sh '''
                    echo "================================"
                    echo "Setting up Ingress Access"
                    echo "================================"
                    
                    # Check if ingress-nginx is installed
                    if ! kubectl get pods -n ingress-nginx &> /dev/null; then
                        echo "Ingress NGINX not found. Installing..."
                        minikube addons enable ingress
                        
                        echo "Waiting for ingress controller to be ready..."
                        kubectl wait --namespace ingress-nginx \
                          --for=condition=ready pod \
                          --selector=app.kubernetes.io/component=controller \
                          --timeout=120s || echo "Ingress controller not ready yet"
                    else
                        echo "Ingress NGINX already installed"
                    fi
                    
                    echo ""
                    echo "================================"
                    echo "Access Information:"
                    echo "================================"
                    MINIKUBE_IP=$(minikube ip)
                    echo "Minikube IP: $MINIKUBE_IP"
                    echo ""
                    echo "To access your application, run:"
                    echo "  curl http://$MINIKUBE_IP"
                    echo ""
                    echo "Or add this to your /etc/hosts:"
                    echo "  $MINIKUBE_IP  my-nginx-app.local"
                    echo "Then access: http://my-nginx-app.local"
                '''
            }
        }
    }

    post {
        success {
            echo '================================'
            echo 'Pipeline executed successfully! ✓'
            echo '================================'
            echo ''
            echo 'Your application has been deployed to Minikube.'
            echo ''
            echo 'Quick access commands:'
            echo '  kubectl get all -n app'
            echo '  kubectl logs -n app -l app=my-nginx-app'
            echo '  minikube service my-nginx-service -n app'
        }
        failure {
            echo '================================'
            echo 'Pipeline failed! ✗'
            echo '================================'
            echo ''
            echo 'Troubleshooting tips:'
            echo '1. Check Minikube status: minikube status'
            echo '2. Check kubectl connectivity: kubectl cluster-info'
            echo '3. View pod logs: kubectl logs -n app -l app=my-nginx-app'
            echo '4. Describe pod: kubectl describe pod -n app -l app=my-nginx-app'
            echo ''
            echo 'Review console output above for detailed error messages.'
        }
        always {
            sh '''
                echo ""
                echo "================================"
                echo "Final Cluster State:"
                echo "================================"
                kubectl get all -n $K8S_NAMESPACE || echo "Could not retrieve resources"
            '''
        }
    }
}