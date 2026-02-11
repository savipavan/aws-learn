pipeline {
    agent any
    tools {
        maven 'Maven 3.9.11'
    }
    environment {
        IMAGE_NAME = "savipavan/myapp"
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKERHUB_CREDENTIALS = credentials('docker-hub-credentials')  // Add in Jenkins Credentials
    }
    stages {
        stage('Checkout Source') {
            steps {
                git branch: 'main', url: 'https://github.com/savipavan/blue-green-jenkins-java-app.git'
            }
        }

        stage('Build Java App') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    // Interactive password input
                    def dockerPassword = input(
                        id: 'dockerPassword', message: 'Enter Docker Hub Password',
                        parameters: [password(defaultValue: '', description: 'Docker Hub Password for savipavan')]
                    )

                    sh """
                        echo '${dockerPassword}' | docker login -u savipavan --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Switch to Docker Desktop') {
            steps {
                script {
                    sh """
                        kubectl config use-context docker-desktop
                        kubectl config current-context
                    """
                }
            }
        }

        stage('Deploy to Docker Desktop') {
            steps {
                script {
                    sh """
                        # Create namespace
                        kubectl create namespace myapp --dry-run=client -o yaml | kubectl apply -f -

                        # Deploy app
                        cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: ${IMAGE_NAME}:${IMAGE_TAG}
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: myapp
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 8085
EOF
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    sh """
                        kubectl -n myapp rollout status deployment/myapp --timeout=300s
                        kubectl -n myapp get pods,svc

                        echo '✅ App deployed successfully!'
                        echo '🌐 Access your app at: http://localhost:8085'
                    """
                }
            }
        }
    }
    post {
        always {
            script {
                sh """
                    docker logout
                    kubectl config use-context docker-desktop  # Ensure stays on docker-desktop
                """
            }
        }
        success {
            echo '🎉 Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
    }
}
