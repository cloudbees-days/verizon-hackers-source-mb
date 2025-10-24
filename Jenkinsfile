// Jenkinsfile — Multibranch-ready, Declarative
// Revisions: replaced cleanWs() -> deleteDir(); replace heredocs with writeFile(); ensure artifacts exist.

pipeline {
  agent none

  options {
    // kept minimal for compatibility
    durabilityHint('MAX_SURVIVABILITY')
    buildDiscarder(logRotator(numToKeepStr: '30'))
    skipDefaultCheckout(true)
    timeout(time: 60, unit: 'MINUTES')
  }

  environment {
    APP_NAME = 'sample-app'
    DOCKER_REGISTRY = 'registry.example.com'
    DOCKER_ORG = 'verizon-demo'
    ARTIFACTORY_CREDENTIALS_ID = 'artifact-repo-creds'
    SIGNING_KEY_ID = 'codesign-key-id'
  }

  parameters {
    choice(name: 'BUILD_KIND', choices: ['container', 'binary'], description: 'Build container image or non-container binary/package')
    booleanParam(name: 'SKIP_SAST', defaultValue: false, description: 'Skip SAST (not recommended)')
    booleanParam(name: 'SKIP_DAST', defaultValue: false, description: 'Skip DAST (main only)')
    booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip unit tests')
  }

  stages {
    stage('Checkout') {
      agent { label 'default' }
      steps {
        checkout scm
        sh 'git rev-parse --abbrev-ref HEAD || true'
      }
    }

    stage('Build') {
      agent { label 'default' }
      stages {

        stage('Setup') {
          steps {
            script {
              env.IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
              echo "IMAGE_TAG=${env.IMAGE_TAG}"
            }
            sh 'mkdir -p reports dist || true'
            writeFile file: 'reports/build-info.txt', text: "build.timestamp=${new Date().time / 1000 as long}\n"
          }
        }

        stage('Compile / Package') {
          steps {
            script {
              if (params.BUILD_KIND == 'container') {
                // Container build placeholder - create dist placeholder so archives don't fail
                sh '''
                  echo "Simulating docker build (placeholder)."
                  mkdir -p dist
                  echo "container image placeholder" > dist/.container-placeholder
                '''
                writeFile file: 'reports/artifact.txt', text: "container:${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG}\n"
              } else {
                sh '''
                  echo "Building non-container artifact (placeholder)."
                  mkdir -p dist
                  echo "hello world" > dist/${APP_NAME}-${IMAGE_TAG}.bin
                  echo "Built mock binary" > dist/README.txt
                '''
                writeFile file: 'reports/artifact.txt', text: "artifact: dist/${APP_NAME}-${IMAGE_TAG}.bin\n"
              }
            }
          }
          post {
            always {
              // archive dist if present; allowEmptyArchive: true avoids hard failure on older Jenkins but we're ensuring dist exists
              archiveArtifacts allowEmptyArchive: true, artifacts: 'dist/**'
            }
          }
        }

        stage('Parallel Checks') {
          parallel {
            stage('Mock Unit Tests') {
              when { expression { !params.SKIP_TESTS } }
              steps {
                script {
                  // write junit xml using writeFile to avoid heredoc whitespace problems
                  writeFile file: 'reports/junit.xml', text: '''<?xml version="1.0" encoding="UTF-8"?>
<testsuite tests="1" failures="0" errors="0" time="0.01">
  <testcase classname="com.example.MockTest" name="mockPass" time="0.01"/>
</testsuite>
'''
                }
              }
              post {
                always {
                  junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                  archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/junit.xml'
                }
              }
            } // Mock Unit Tests

            stage('Lint / Static Checks') {
              steps {
                sh 'mkdir -p reports; echo "Mock linter output" > reports/lint.txt'
                archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/lint.txt'
              }
            } // Lint

            stage('SAST (produce SARIF)') {
              when { expression { !params.SKIP_SAST } }
              steps {
                script {
                  writeFile file: 'reports/sarif.json', text: '''{
  "version": "2.1.0",
  "runs": [
    {
      "tool": { "driver": { "name": "mock-sast", "informationUri": "https://example.com/mock-sast" } },
      "results": []
    }
  ]
}
'''
                }
              }
              post {
                always {
                  archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/sarif.json'
                }
              }
            } // SAST
          } // parallel
        } // Parallel Checks

        stage('Sign & Package') {
          steps {
            script {
              if (params.BUILD_KIND == 'container') {
                // mock sign info
                writeFile file: 'reports/signature.txt', text: "signature:mock-signature\n"
                writeFile file: 'reports/artifact.txt', text: "container:${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG}\n"
              } else {
                writeFile file: 'reports/signature.txt', text: "signature:mock-signature\n"
                writeFile file: 'reports/artifact.txt', text: "artifact: dist/${APP_NAME}-${IMAGE_TAG}.bin\n"
              }
            }
            archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**'
            fingerprint 'reports/**'
          }
        } // Sign & Package

        stage('Push Artifact (non-PR)') {
          when { not { changeRequest() } }
          steps {
            script {
              if (params.BUILD_KIND == 'container') {
                // ensure we create upload info even for container case
                writeFile file: 'reports/upload.info', text: "Pushed container (mock): ${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG}\n"
                sh 'echo "Pushing container to registry (mock). Replace with docker login/push."'
              } else {
                withCredentials([usernamePassword(credentialsId: env.ARTIFACTORY_CREDENTIALS_ID, usernameVariable: 'ART_USER', passwordVariable: 'ART_PSW')]) {
                  writeFile file: 'reports/upload.info', text: "Uploaded dist/${APP_NAME}-${IMAGE_TAG}.bin (mock) by ${ART_USER}\n"
                  sh 'echo "Uploading artifact to repo (mock)."'
                }
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/upload.info'
            }
          }
        } // Push Artifact

        stage('Publish Build Metadata') {
          steps {
            script {
              writeFile file: 'reports/sbom.json', text: '{ "sbom": "mock", "components": [] }'
              writeFile file: 'reports/build-metadata.json', text: "{ \"image\": \"${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG}\" }\n"
            }
            archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**'
          }
        } // Publish Build Metadata

      } // nested Build stages
    } // Build

    stage('Dev') {
      agent { label 'default' }
      when {
        allOf {
          anyOf {
            branch pattern: "feature/.*", comparator: "REGEXP"
            not { branch 'main' }
          }
        }
      }
      steps {
        echo 'Dev Stage: simulate deploy to dev environment.'
        sh 'mkdir -p reports; echo "dev-deploy:ok" > reports/dev-deploy.txt'
        archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/dev-deploy.txt'
      }
    }

    stage('Test') {
      agent { label 'default' }
      steps {
        echo 'Test Stage: run integration tests and optional DAST'
        script {
          if (env.BRANCH_NAME == 'main' && !params.SKIP_DAST) {
            writeFile file: 'reports/dast.json', text: '{ "dast": "mock-result" }'
            sh 'echo "Running DAST (mock) against test env - replace with ZAP / Arachni etc."'
            archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/dast.json'
          } else {
            echo "Skipping DAST (branch != main or SKIP_DAST)"
          }
        }
      }
    }
stage('Prod') {
  agent any
  when { beforeAgent true; branch 'main' }
  steps {
    timeout(time: 30, unit: 'MINUTES') {
      input message: "Approve promotion of ${APP_NAME}:${IMAGE_TAG} to PROD?",
            ok: 'Approve',
            submitter: 'approver1,approver2'
    }
    echo 'Prod Stage: deploy approved artifact to Prod (mock)'
    sh 'mkdir -p reports; echo "prod-deploy:ok" > reports/prod-deploy.txt'
    archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/prod-deploy.txt'
  }
}

  } // stages

  post {
    success { echo "Pipeline succeeded on branch ${env.BRANCH_NAME}" }
    failure { echo "Pipeline failed on branch ${env.BRANCH_NAME}" }
    always {
      // deleteDir() is safer / available in your Jenkins instance
      script { deleteDir() }
    }
  }
}
