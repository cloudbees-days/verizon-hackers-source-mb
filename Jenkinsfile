pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Build Info') {
            steps {
                echo 'Build Information:'
                echo "Branch: ${env.BRANCH_NAME}"
                echo "Build Number: ${env.BUILD_NUMBER}"
                sh 'pwd'
                sh 'ls -la'
            }
        }
        
        stage('Check Node') {
            steps {
                echo 'Checking Node.js availability...'
                script {
                    try {
                        sh 'node --version'
                        sh 'npm --version'
                    } catch (Exception e) {
                        echo 'Node.js not available, skipping npm steps'
                    }
                }
            }
        }
        
        stage('Success') {
            steps {
                echo 'Pipeline completed successfully!'
                echo 'Ready for CloudBees Platform integration testing'
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline finished!'
        }
        success {
            echo 'Build succeeded - multibranch pipeline is working!'
        }
        failure {
            echo 'Build failed - check logs for details'
        }
    }
}