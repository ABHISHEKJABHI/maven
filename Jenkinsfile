pipeline {
    agent any
    
    // environment {
        // PATH = "/opt/apache-maven-3.8.8/bin/:$PATH"
    // }
tools
{ maven ('maven')
     }
     options {
        timeout(time: 10, unit:'MINUTES')
    }
    stages {
        stage('git clone') {
            steps {
                git branch: 'master', url: 'https://github.com/ABHISHEKJABHI/maven.git'
            }
        }
        
        stage('mvn') {
            steps {
                sh "mvn clean package"
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml' 
                }
            }
        }
    }
}