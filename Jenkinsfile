pipeline {
    agent any
    
    stages {
        stage('Test') {
            steps {
                echo 'Multibranch Pipeline Working!'
                echo "Branch: ${env.BRANCH_NAME}"
                echo "Build: ${env.BUILD_NUMBER}"
            }
        }
        
        stage('Validate') {
            steps {
                echo 'CloudBees Platform Integration Ready!'
                sh 'echo "Success from WesTestMB"'
            }
        }
    }
}