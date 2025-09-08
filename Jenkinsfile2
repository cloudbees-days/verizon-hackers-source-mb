pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: node
                    image: node:lts-alpine
                    command:
                    - sleep
                    args:
                    - 99d
            '''
        }
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                container('node') {
                    sh 'npm ci'
                }
            }
        }

        stage('Build') {
            steps {
                container('node') {
                    sh 'npm run build'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'dist/**/*', allowEmptyArchive: true
                }
            }
        }

        stage('Test') {
            steps {
                container('node') {
                    sh 'npm run test:unit -- --reporters=default --reporters=jest-junit'
                }
            }
            post {
                always {
                    junit '**/junit.xml'
                }
            }
        }

        stage('Register Build Artifact') {
            steps {
                registerBuildArtifactMetadata(
                    name: "hackers-organized",
                    url: "${BUILD_URL}artifact/dist/",
                    version: "2.1-${BUILD_NUMBER}",
                    digest: "${GIT_COMMIT}",
                    label: "demo",
                    type: "app"
                )
            }
        }
    }
}
