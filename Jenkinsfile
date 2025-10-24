// Jenkinsfile — Multibranch-ready, Declarative
// Keeps your top-level stages (Build / Dev / Test / Prod) while adding:
// - parallel unit tests + mock junit report
// - SARIF generation placeholder for SAST
// - artifact signing & packaging placeholders
// - conditional DAST on main
// - push artifact (skipped for PRs)
// - archival/fingerprinting for Unify ingestion

pipeline {
  agent none

  options {
    ansiColor('xterm')
    timestamps()
    skipDefaultCheckout(true)
    buildDiscarder(logRotator(numToKeepStr: '30'))
    timeout(time: 60, unit: 'MINUTES')
  }

  // Lightweight environment - store credential IDs here and resolve with withCredentials() where needed
  environment {
    APP_NAME = 'sample-app'
    DOCKER_REGISTRY = 'registry.example.com'
    DOCKER_ORG = 'verizon-demo'
    ARTIFACTORY_CREDENTIALS_ID = 'artifact-repo-creds'  // update to your creds id
    SIGNING_KEY_ID = 'codesign-key-id'                   // update to your creds id (or secret text)
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
        // Multibranch checkout
        checkout scm
        // Ensure a clean-ish workspace for deterministic packaging
        sh 'git rev-parse --abbrev-ref HEAD || true'
      }
    }

    stage('Build') {
      agent { label 'default' }

      // Nested stages inside Build for clarity
      stages {
        stage('Setup') {
          steps {
            script {
              // In Jenkins declarative, BRANCH_NAME is available at runtime.
              env.IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
              echo "IMAGE_TAG=${env.IMAGE_TAG}"
            }
            sh '''
              mkdir -p reports
              echo "build.timestamp=$(date +%s)" > reports/build-info.txt
            '''
          }
        }

        stage('Compile / Package') {
          steps {
            script {
              if (params.BUILD_KIND == 'container') {
                sh '''
                  echo "Simulating docker build (placeholder). Replace with your docker build/push steps if node has docker."
                  # docker build -t ${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG} .
                '''
              } else {
                sh '''
                  echo "Building non-container artifact (placeholder)."
                  mkdir -p dist
                  echo "hello world" > dist/${APP_NAME}-${IMAGE_TAG}.bin
                  echo "Built mock binary" > dist/README.txt
                '''
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'dist/**'
            }
          }
        }

        stage('Parallel Checks') {
          parallel {
            stage('Mock Unit Tests') {
              when { expression { !params.SKIP_TESTS } }
              steps {
                sh '''
                  set -e
                  mkdir -p reports
                  # Create a tiny JUnit XML (1 passing test)
                  cat > reports/junit.xml <<'XML'
                  <?xml version="1.0" encoding="UTF-8"?>
                  <testsuite tests="1" failures="0" errors="0" time="0.01">
                    <testcase classname="com.example.MockTest" name="mockPass" time="0.01"/>
                  </testsuite>
                  XML
                  echo "Mock unit test complete"
                '''
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
                sh '''
                  mkdir -p reports
                  echo "Mock linter output" > reports/lint.txt
                '''
                archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/lint.txt'
              }
            } // Lint

            stage('SAST (produce SARIF)') {
              when {
                allOf {
                  expression { !params.SKIP_SAST }
                }
              }
              steps {
                sh '''
                  mkdir -p reports
                  # Produce a minimal SARIF file so Unify can ingest it as a placeholder.
                  cat > reports/sarif.json <<'JSON'
                  {
                    "version": "2.1.0",
                    "runs": [
                      {
                        "tool": {
                          "driver": { "name": "mock-sast", "informationUri": "https://example.com/mock-sast" }
                        },
                        "results": []
                      }
                    ]
                  }
                  JSON
                  echo "Mock SARIF produced"
                '''
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
              // Produce a "signed" artifact (mock). In real usage, call cosign/gpg etc
              if (params.BUILD_KIND == 'container') {
                sh '''
                  echo "Mock: sign container image (replace with cosign or registry-signing step)."
                  mkdir -p reports
                  echo "container:${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG}" > reports/artifact.txt
                  echo "signature:mock-signature" > reports/signature.txt
                '''
              } else {
                sh '''
                  mkdir -p reports
                  echo "artifact: dist/${APP_NAME}-${IMAGE_TAG}.bin" > reports/artifact.txt
                  echo "signature:mock-signature" > reports/signature.txt
                '''
              }
            }
            archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**'
            fingerprint 'reports/**'
          }
        } // Sign & Package

        stage('Push Artifact (non-PR)') {
          when { not { changeRequest() } } // skip for PRs
          steps {
            script {
              if (params.BUILD_KIND == 'container') {
                sh '''
                  echo "Pushing container to registry (mock). Replace with docker login/push."
                '''
              } else {
                withCredentials([usernamePassword(credentialsId: env.ARTIFACTORY_CREDENTIALS_ID, usernameVariable: 'ART_USER', passwordVariable: 'ART_PSW')]) {
                  sh '''
                    echo "Uploading artifact to repo (mock). Use curl/Artifactory CLI in real pipeline."
                    echo "USER=${ART_USER}" > reports/upload.info
                    echo "Uploaded dist/${APP_NAME}-${IMAGE_TAG}.bin (mock)" >> reports/upload.info
                  '''
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
            sh '''
              mkdir -p reports
              # Mock SBOM and metadata for Unify
              cat > reports/sbom.json <<'SBOM'
              { "sbom": "mock", "components": [] }
              SBOM
              echo "{ \"image\": \"${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG}\" }" > reports/build-metadata.json || true
              echo "Published build metadata"
            '''
            archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**'
          }
        } // Publish Build Metadata

      } // nested stages of Build
    } // Build

    // Keep your Dev/Test/Prod stage names — add real deploy/test logic inside them
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
        sh '''
          echo "Deploying ${APP_NAME}:${IMAGE_TAG} to dev (mock)"
          mkdir -p reports
          echo "dev-deploy:ok" > reports/dev-deploy.txt
        '''
        archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/dev-deploy.txt'
      }
    }

    stage('Test') {
      agent { label 'default' }
      steps {
        echo 'Test Stage: run integration tests and optional DAST'
        // DAST typically runs against a deployed test environment; we only run it for main by default
        script {
          if (env.BRANCH_NAME == 'main' && !params.SKIP_DAST) {
            sh '''
              echo "Running DAST (mock) against test env - replace with ZAP / Arachni etc."
              mkdir -p reports
              echo "{ \\"dast\\": \\"mock-result\\" }" > reports/dast.json
            '''
            archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/dast.json'
          } else {
            echo "Skipping DAST (branch != main or SKIP_DAST)"
          }
        }
      }
    }

    stage('Prod') {
      agent { label 'default' }
      when { branch 'main' }
      steps {
        script {
          // Manual gate for production promotion — Unify Edition 2 should orchestrate promotions in production scenarios.
          timeout(time: 30, unit: 'MINUTES') {
            input message: "Approve promotion of ${APP_NAME}:${IMAGE_TAG} to PROD?", ok: 'Approve', submitter: 'approver1,approver2'
          }
          echo 'Prod Stage: deploy approved artifact to Prod (mock)'
          sh '''
            mkdir -p reports
            echo "prod-deploy:ok" > reports/prod-deploy.txt
          '''
          archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/prod-deploy.txt'
        }
      }
    }

  } // stages

  post {
    success {
      echo "Pipeline succeeded on branch ${env.BRANCH_NAME}"
    }
    failure {
      echo "Pipeline failed on branch ${env.BRANCH_NAME}"
    }
    always {
      // Publish everything we archived already, and try to clean workspace
      cleanWs notFailBuild: true
    }
  }
}
