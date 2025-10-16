pipeline {
    agent any
    triggers {
        githubPush()
    }
    environment {
        DOCKER_IMAGE = "gani220/mart-app-doc-img"
        DOCKER_TAG   = "${BUILD_NUMBER}"
        EKS_CLUSTER_NAME = "mart-test-cluster"
        AWS_REGION = "mx-central-1"
    }
    stages { 
        stage('Checkout') {
            steps {
                git credentialsId: 'github_credentials',
                    branch: 'master',
                    url: 'https://github.com/VeeraboinaSaiGanesh/Mart-App-Kubernetes.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "🛠️ Building Docker image..."
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                    docker.build("${DOCKER_IMAGE}:latest")
                }
            }
        }

        

        stage('Push Docker Image') {
            steps {
                script {
                    echo "📦 Pushing Docker image..."
                    docker.withRegistry('https://index.docker.io/v1/', 'Docker_Credentials') {
                        sh """
                            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                    echo "✅ Docker images pushed successfully"
                }
            }
        }

        stage('Deploy  Kubernetes') {
            steps {
                withAWS(credentials: 'G_AWS_CRED', region: "${AWS_REGION}") {
                    script {
                        sh """
                            echo "🔄 Updating kubeconfig..."
                            aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}

                            echo "🚀 Deploying to Kubernetes..."
                            # Deploy MongoDB first
                            kubectl apply -f mongodb-deployment.yaml

                            # Update application image and deploy
                            kubectl apply -f app-deployment.yaml
                            #kubectl set image deployment/fitness-tracker-app \\
                                #fitness-tracker-app=${DOCKER_IMAGE}:${DOCKER_TAG} --record
                            

                            echo "⏳ Waiting for deployments to complete..."
                            kubectl rollout status deployment/mongodb --timeout=300s
                            kubectl rollout status deployment/mart-app --timeout=300s

                            echo "📊 Deployment status:"
                            kubectl get deployments
                            kubectl get services
                            kubectl get pods
                        """
                    }
                }
            }
        }

        stage('Get LoadBalancer URL') {
            steps {
                withAWS(credentials: 'G_AWS_CRED', region: "${AWS_REGION}") {
                    script {
                        sh """
                            echo "🌐 Getting LoadBalancer URL..."
                            aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}

                            i=1
                            while [ \$i -le 10 ]; do
                                EXTERNAL_IP=\$(kubectl get service mart-app-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || echo "")
                                EXTERNAL_HOSTNAME=\$(kubectl get service mart-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo "")

                                if [ ! -z "\$EXTERNAL_IP" ]; then
                                    echo "🌐 Application URL: http://\$EXTERNAL_IP"
                                    break
                                elif [ ! -z "\$EXTERNAL_HOSTNAME" ]; then
                                    echo "🌐 Application URL: http://\$EXTERNAL_HOSTNAME"
                                    break
                                fi

                                echo "⏳ Waiting for LoadBalancer... attempt \$i/10"
                                i=\$((i+1))
                                sleep 20
                            done

                            # Show final service status
                            kubectl get service mart-app-service
                            echo "✅ Deployment completed successfully!"
                        """
                    }
                }
            }
        }
    }
    post {
        success {
            echo "✅ Deployment successful!"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}


