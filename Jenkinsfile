pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node26'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'

        BACKEND_IMAGE = 'anjalia123/project-backend:latest'
        FRONTEND_IMAGE = 'anjalia123/project-frontend:latest'
        ADMIN_IMAGE = 'anjalia123/project-admin:latest'

        EKS_CLUSTER_NAME = 'my-eks'
        AWS_REGION = 'ap-south-1'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/AnjaliKumariPrasad/cloud-native-automation-platform.git'

                sh 'ls -la'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                          -Dsonar.projectName=Cloud-Native-Automation-Platform \
                          -Dsonar.projectKey=Cloud-Native-Automation-Platform
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate(
                        abortPipeline: false,
                        credentialsId: 'Sonar-token'
                    )
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "Installing frontend dependencies..."
                    cd app/frontend
                    npm ci

                    echo "Installing backend dependencies..."
                    cd ../backend
                    npm ci

                    echo "Installing admin dependencies..."
                    cd ../admin
                    npm ci
                '''
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh '''
                    trivy fs . > trivyfs.txt
                '''
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(
                        credentialsId: 'docker'
                    ) {
                        sh '''
                            echo "Building backend image..."
                            docker build -t $BACKEND_IMAGE ./app/backend

                            echo "Building frontend image..."
                            docker build -t $FRONTEND_IMAGE ./app/frontend

                            echo "Building admin image..."
                            docker build -t $ADMIN_IMAGE ./app/admin


                            echo "Pushing backend image..."
                            docker push $BACKEND_IMAGE

                            echo "Pushing frontend image..."
                            docker push $FRONTEND_IMAGE

                            echo "Pushing admin image..."
                            docker push $ADMIN_IMAGE
                        '''
                    }
                }
            }
        }

        stage('Deploy to EKS Cluster') {
            steps {
                sh '''
                    echo "Verifying AWS credentials..."
                    aws sts get-caller-identity

                    echo "Configuring kubectl for EKS..."
                    aws eks update-kubeconfig \
                        --name $EKS_CLUSTER_NAME \
                        --region $AWS_REGION

                    echo "Deploying to Kubernetes..."
                    kubectl apply -f Kubernetes/deployment.yml
                    kubectl apply -f Kubernetes/service.yml

                    echo "Checking Pods..."
                    kubectl get pods

                    echo "Checking Services..."
                    kubectl get svc
                '''
            }
        }
    }

    post {
        always {
            emailext(
                attachLog: true,
                subject: "${currentBuild.currentResult}: ${env.JOB_NAME}",
                body: """
                    Project: ${env.JOB_NAME}<br/>
                    Build Number: ${env.BUILD_NUMBER}<br/>
                    Build URL: ${env.BUILD_URL}<br/>
                """,
                to: 'anjali97067@gmail.com',
                attachmentsPattern: 'trivyfs.txt'
            )
        }
    }
}