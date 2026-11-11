
pipeline {
    agent any

    environment {
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '135692633090'
        ECR_REPOSITORY = 'hostelmanagement'
        ECR_REGISTRY   = '135692633090.dkr.ecr.us-east-1.amazonaws.com'
        IMAGE_URI      = '135692633090.dkr.ecr.us-east-1.amazonaws.com/hostelmanagement'

        EKS_CLUSTER    = 'devops-cluster'
        HELM_RELEASE   = 'hostelmanagement'
        K8S_NAMESPACE  = 'default'
        HELM_CHART     = 'helm/hostelmanagement'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/Kalpeshpatilblaze/HostelManagment1.git'
            }
        }

        stage('Maven Build') {
            steps {
                sh '''
                    echo "Building Java application..."
                    mvn clean package
                    ls -lh target/HostelManagementSystem.war
                '''
            }
        }

        stage('Archive WAR') {
            steps {
                archiveArtifacts artifacts: 'target/*.war',
                                 fingerprint: true
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "Building Docker image..."

                    docker build \
                      -t $IMAGE_URI:$BUILD_NUMBER .
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    echo "Logging into Amazon ECR..."

                    aws ecr get-login-password \
                      --region $AWS_REGION |
                    docker login \
                      --username AWS \
                      --password-stdin $ECR_REGISTRY
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    echo "Pushing Docker image to ECR..."

                    docker push $IMAGE_URI:$BUILD_NUMBER
                '''
            }
        }

        stage('Configure EKS') {
            steps {
                sh '''
                    echo "Configuring Kubernetes access..."

                    aws eks update-kubeconfig \
                      --region $AWS_REGION \
                      --name $EKS_CLUSTER

                    kubectl get nodes
                '''
            }
        }

        stage('Helm Lint') {
            steps {
                sh '''
                    helm lint $HELM_CHART
                '''
            }
        }

        stage('Deploy using Helm') {
            steps {
                sh '''
                    echo "Deploying application to EKS..."

                    helm upgrade --install $HELM_RELEASE $HELM_CHART \
                      --namespace $K8S_NAMESPACE \
                      --create-namespace \
                      --set image.repository=$IMAGE_URI \
                      --set image.tag=$BUILD_NUMBER \
                      --wait \
                      --timeout 5m
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Checking Helm release..."

                    helm status $HELM_RELEASE \
                      --namespace $K8S_NAMESPACE

                    echo "Checking Kubernetes deployment..."

                    kubectl rollout status \
                      deployment/$HELM_RELEASE \
                      -n $K8S_NAMESPACE \
                      --timeout=300s

                    kubectl get pods \
                      -n $K8S_NAMESPACE -o wide

                    kubectl get services \
                      -n $K8S_NAMESPACE
                '''
            }
        }
    }

    post {
        success {
            echo 'Hostel Management application successfully deployed to Amazon EKS using Helm!'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins console output.'
        }

        always {
            sh '''
                docker logout $ECR_REGISTRY || true
            '''
            cleanWs()
        }
    }
}

