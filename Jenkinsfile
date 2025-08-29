pipeline {
    agent any
    
    environment {
        // Update PATH if needed
        PATH = "/opt/apache-maven-3.8.8/bin/:$PATH"
         JAVA_HOME = "/usr/lib/jvm/java-21-openjdk-amd64"
         PATH = "${env.JAVA_HOME}/bin:${env.PATH}"
        
        // Docker and Kubernetes configuration
        DOCKER_REGISTRY = "docker.io/abhishek7483/image-name:tag"  // e.g., docker.io, gcr.io, ecr.amazonaws.com
        DOCKER_IMAGE_NAME = "my-app"
        DOCKER_IMAGE_TAG = "${env.BUILD_ID}"
        KUBE_NAMESPACE = "your-namespace"
        KUBE_CONFIG = "/path/to/kubeconfig"  // Path to kubeconfig file
    }
    
    tools {
        maven 'mvn'
        jdk 'openjdk 21'  // Specify your JDK version
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    
    stages {
        stage('Git Clone') {
            steps {
                git branch: 'master', 
                     url: 'https://github.com/ABHISHEKJABHI/maven.git',
                     credentialsId: 'your-git-credentials'  // If private repo
            }
        }
        
        stage('Maven Build') {
            steps {
                sh "mvn clean package -DskipTests"
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    // Create Dockerfile if it doesn't exist
                    def dockerfileContent = """
                    FROM openjdk:11-jre-slim
                    WORKDIR /app
                    COPY target/*.jar app.jar
                    EXPOSE 8080
                    ENTRYPOINT ["java", "-jar", "app.jar"]
                    """
                    
                    writeFile file: 'Dockerfile', text: dockerfileContent
                    
                    // Build Docker image
                    sh "docker build -t ${env.DOCKER_IMAGE_NAME}:${env.DOCKER_IMAGE_TAG} ."
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    // Login to Docker registry (example for Docker Hub)
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        usernameVariable: 'abhishek7483',
                        passwordVariable: 'abhi7483'
                    )]) {
                        sh "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin ${env.DOCKER_REGISTRY}"
                    }
                    
                    // Tag and push image
                    sh "docker tag ${env.DOCKER_IMAGE_NAME}:${env.DOCKER_IMAGE_TAG} ${env.DOCKER_REGISTRY}/${env.DOCKER_IMAGE_NAME}:${env.DOCKER_IMAGE_TAG}"
                    sh "docker push ${env.DOCKER_REGISTRY}/${env.DOCKER_IMAGE_NAME}:${env.DOCKER_IMAGE_TAG}"
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Create Kubernetes deployment manifest
                    def deploymentYaml = """
                    apiVersion: apps/v1
                    kind: Deployment
                    metadata:
                      name: ${env.DOCKER_IMAGE_NAME}
                      namespace: ${env.KUBE_NAMESPACE}
                    spec:
                      replicas: 2
                      selector:
                        matchLabels:
                          app: ${env.DOCKER_IMAGE_NAME}
                      template:
                        metadata:
                          labels:
                            app: ${env.DOCKER_IMAGE_NAME}
                        spec:
                          containers:
                          - name: ${env.DOCKER_IMAGE_NAME}
                            image: ${env.DOCKER_REGISTRY}/${env.DOCKER_IMAGE_NAME}:${env.DOCKER_IMAGE_TAG}
                            ports:
                            - containerPort: 8080
                            resources:
                              requests:
                                memory: "256Mi"
                                cpu: "250m"
                              limits:
                                memory: "512Mi"
                                cpu: "500m"
                    ---
                    apiVersion: v1
                    kind: Service
                    metadata:
                      name: ${env.DOCKER_IMAGE_NAME}-service
                      namespace: ${env.KUBE_NAMESPACE}
                    spec:
                      selector:
                        app: ${env.DOCKER_IMAGE_NAME}
                      ports:
                      - port: 80
                        targetPort: 8080
                      type: LoadBalancer
                    """
                    
                    writeFile file: 'deployment.yaml', text: deploymentYaml
                    
                    // Deploy to Kubernetes
                    withEnv(["KUBECONFIG=${env.KUBE_CONFIG}"]) {
                        sh "kubectl apply -f deployment.yaml"
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    withEnv(["KUBECONFIG=${env.KUBE_CONFIG}"]) {
                        // Wait for deployment to be ready
                        sh "kubectl rollout status deployment/${env.DOCKER_IMAGE_NAME} -n ${env.KUBE_NAMESPACE} --timeout=300s"
                        
                        // Check pods status
                        sh "kubectl get pods -n ${env.KUBE_NAMESPACE} -l app=${env.DOCKER_IMAGE_NAME}"
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
            // Optional: Send notification
        }
        failure {
            echo 'Pipeline failed!'
            // Optional: Send failure notification
        }
        always {
            // Cleanup
            sh 'docker logout ${env.DOCKER_REGISTRY}'
            echo 'Pipeline execution completed'
        }
    }
}
