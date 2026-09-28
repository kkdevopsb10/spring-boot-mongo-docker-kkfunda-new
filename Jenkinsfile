pipeline {
    agent any

    stages {

        stage('Checkout from GitHub') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/kkdevopsb10/spring-boot-mongo-docker-kkfunda-new.git'
            }
        }

        stage('Setup KubeConfig') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-eks-cred'
                ]]) {
                    sh '''
                        aws eks update-kubeconfig \
                            --region ap-south-1 \
                            --name my-cluster

                        kubectl cluster-info
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-eks-cred'
                ]]) {
                    sh '''
                        kubectl apply -f springBootMongo.yml --validate=false
                    '''
                }
            }
        }

        stage('Wait for Pods and Services') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-eks-cred'
                ]]) {
                    sh '''
                        echo "Waiting for Kubernetes resources..."

                        TIMEOUT=60
                        INTERVAL=5
                        ELAPSED=0

                        while [ $ELAPSED -lt $TIMEOUT ]; do

                            echo "----------------------------------------"
                            echo "Checking pods..."
                            kubectl get pods

                            echo "Checking services..."
                            kubectl get svc

                            # Check whether all pods are Running
                            PODS_NOT_RUNNING=$(kubectl get pods --no-headers 2>/dev/null | \
                                awk '$3 != "Running" {count++} END {print count+0}')

                            # Check whether all Running pods are Ready
                            PODS_NOT_READY=$(kubectl get pods --no-headers 2>/dev/null | \
                                awk '$3 == "Running" {
                                    split($2, ready, "/")
                                    if (ready[1] != ready[2]) count++
                                } END {print count+0}')

                            # Check that at least one service exists
                            SERVICE_COUNT=$(kubectl get svc --no-headers 2>/dev/null | wc -l)

                            if [ "$PODS_NOT_RUNNING" -eq 0 ] && \
                               [ "$PODS_NOT_READY" -eq 0 ] && \
                               [ "$SERVICE_COUNT" -gt 0 ]; then

                                echo "========================================"
                                echo "SUCCESS: Pods are Running and Ready."
                                echo "SUCCESS: Kubernetes Service exists."
                                echo "========================================"

                                kubectl get pods
                                kubectl get svc

                                exit 0
                            fi

                            echo "Resources are not ready yet..."
                            echo "Waiting 5 seconds..."
                            sleep $INTERVAL

                            ELAPSED=$((ELAPSED + INTERVAL))
                        done

                        echo "========================================"
                        echo "ERROR: Kubernetes resources did not become"
                        echo "ready within 1 minute."
                        echo "========================================"

                        kubectl get pods
                        kubectl get svc

                        exit 1
                    '''
                }
            }
        }
    }
}
