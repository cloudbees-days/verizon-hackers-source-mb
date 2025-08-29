pipeline {
    agent {
        label 'default'
    }
    
    stages {
        stage('Hello World') {
            steps {
                echo 'Hello from WesTestMB!'
                echo "Branch: ${env.BRANCH_NAME}"
                echo "Build: ${env.BUILD_NUMBER}"
            }
        }
        
        stage('Environment') {
            steps {
                echo 'Environment check:'
                sh 'whoami'
                sh 'pwd'
            }
        }
        
        stage('Success') {
            steps {
                echo 'Multibranch pipeline is working!'
                echo 'Ready for CloudBees Platform integration!'
            }
        }
    }
}