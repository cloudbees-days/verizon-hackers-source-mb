// Jenkinsfile — fixes: deleteDir() instead of cleanWs(), guarded credentials usage.
// Keeps your original stages/flow.

pipeline {
  agent any

  options {
    // timestamps()/ansiColor removed to avoid plugin/option errors
    durabilityHint('MAX_SURVIVABILITY')
    buildDiscarder(logRotator(numToKeepStr: '30'))
    skipDefaultCheckout(true)
    timeout(time: 60, unit: 'MINUTES')
  }

  environment {
    APP_NAME        = 'sample-app'
    DOCKER_REGISTRY = 'registry.example.com'
    DOCKER_ORG      = 'verizon-demo'
    IMAGE_TAG       = "${env.BRANCH_NAME ?: 'unknown'}-${env.BUILD_NUMBER}"
    // NOTE: removed credentials(...) from environment to avoid immediate failures
    // Use withCredentials(...) in the stages below (guarded by try/catch)
    RUN_DAST        = "${env.BRANCH_NAME == 'main' ? 'true' : 'false'}"
  }

  parameters {
    booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip unit/integration tests')
    booleanParam(name: 'SKIP_SAST', defaultValue: false, description: 'Skip SAST (not recommended)')
    booleanParam(name: 'SKIP_DAST', defaultValue: false, description: 'Skip DAST on main')
    choice(name: 'BUILD_KIND', choices: ['container', 'binary'], description: 'Build container image or non-container binary/package')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git fetch --prune --unshallow || true'
      }
    }

    stage('Prepare / Tooling') {
      steps {
        sh '''
          echo "Node/Java/Go/etc setup goes here if you use tool installers"
          java -version || true
          node -v || true
          go version || true
        '''
      }
    }

    stage('Static Analysis & Tests (parallel)') {
      parallel {
        stage('Lint') {
          when { expression { !params.SKIP_TESTS } }
          steps {
            sh '''
              echo "Run linters here (eslint, golangci-lint, flake8, etc.)"
            '''
          }
        }

        stage('Unit Tests') {
          when { expression { !params.SKIP_TESTS } }
          steps {
            sh '''
              mkdir -p reports/junit
              # Example JUnit files (replace with real test runner output)
              cat > reports/junit/TEST-add.xml <<'XML'
              <?xml version="1.0" encoding="UTF-8"?>
              <testsuite tests="1" failures="0" name="add-suite">
                <testcase classname="sample.calculator" name="test_add" time="0.01">
                  <system-out>input=(3,4); expected=7; actual=7</system-out>
                </testcase>
              </testsuite>
              XML
            '''
          }
          post {
            always {
              junit allowEmptyResults: true, testResults: 'reports/junit/*.xml'
              archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**'
            }
          }
        }

        stage('SAST') {
          when {
            allOf {
              expression { !params.SKIP_SAST }
              not { changeRequest() }
            }
          }
          steps {
            sh '''
              mkdir -p reports/sarif
              echo '{ "version":"2.1.0", "runs":[ { "tool": { "driver": { "name":"mock-sast" } }, "results": [] } ] }' > reports/sarif/result.sarif
            '''
          }
          post { always { archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**' } }
        }
      }
    } // parallel

    stage('Build') {
      steps {
        script {
          if (params.BUILD_KIND == 'container') {
            sh """
              echo "Building container image ${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG}"
              docker build -t ${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG} .
            """
          } else {
            sh '''
              echo "Building non-container binary/package"
              mkdir -p dist
              echo "hello" > dist/app-binary
            '''
          }
        }
      }
    }

    stage('Sign & Package') {
      steps {
        script {
          // Try to use codesign credential only if present; fail-safe if missing.
          try {
            withCredentials([string(credentialsId: 'codesign-key-id', variable: 'CODESIGN_KEY')]) {
              if (params.BUILD_KIND == 'container') {
                sh '''
                  echo "Sign image (placeholder) using CODESIGN_KEY"
                  # cosign sign --key env://CODESIGN_KEY ${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG} || true
                '''
              } else {
                sh '''
                  mkdir -p dist/signature
                  echo "signed-binary:${IMAGE_TAG}" > dist/signature/${APP_NAME}.sig
                  # gpg --batch --yes --detach-sign --armor -u "$CODESIGN_KEY" dist/app-binary || true
                '''
                archiveArtifacts artifacts: 'dist/**', fingerprint: true
              }
            }
          } catch (err) {
            // Credentials not found or signing failed; continue but warn
            echo "Signing skipped or failed (codesign-key-id missing or error): ${err}"
            // Create a placeholder signature so downstream can continue
            sh 'mkdir -p dist/signature && echo "unsigned:${IMAGE_TAG}" > dist/signature/${APP_NAME}.sig || true'
            archiveArtifacts artifacts: 'dist/**', fingerprint: true
          }
        }
      }
    }

    stage('Push Artifact/Image') {
      when { not { changeRequest() } }
      steps {
        script {
          try {
            // Try to find artifact-repo-creds (username/password). If missing, withCredentials will throw and we catch it.
            withCredentials([usernamePassword(credentialsId: 'artifact-repo-creds', usernameVariable: 'ART_USER', passwordVariable: 'ART_PSW')]) {
              if (params.BUILD_KIND == 'container') {
                sh """
                  echo "Logging in (mock) and pushing image..."
                  echo "$ART_PSW" | docker login ${DOCKER_REGISTRY} -u "$ART_USER" --password-stdin || true
                  # docker push ${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG} || true
                """
              } else {
                sh '''
                  echo "Upload binary to artifact repo (mock). Use $ART_USER and $ART_PSW"
                  # curl -u "$ART_USER:$ART_PSW" -T dist/${APP_NAME} https://myrepo/repo/...
                '''
              }
            }
          } catch (err) {
            echo "artifact-repo-creds not available or push failed; skipping push. Error: ${err}"
          }
        }
      }
    }

    stage('DAST (main only)') {
      when {
        allOf {
          branch 'main'
          expression { params.RUN_DAST == 'true' && !params.SKIP_DAST }
        }
      }
      steps {
        sh '''
          echo "Run DAST here against a test endpoint (placeholder)"
          # zap-baseline.py -t https://test-env.example.com -J reports/zap.json || true
        '''
      }
      post { always { archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**' } }
    }

    stage('Publish Build Metadata (for Unify)') {
      steps {
        sh '''
          echo "Emit SARIF / JUnit / SBOM / signature metadata for Unify ingestion"
          mkdir -p reports/metadata
          echo "{ \\"artifact\\": \\"${APP_NAME}\\", \\"tag\\": \\"${IMAGE_TAG}\\" }" > reports/metadata/build.json
        '''
        archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**'
      }
    }

    stage('Deploy to Dev (optional)') {
      when { branch pattern: "feature/.*", comparator: "REGEXP" }
      steps {
        sh '''
          echo "Example: helm upgrade --install ${APP_NAME} charts/${APP_NAME} --namespace dev --set image.tag='${IMAGE_TAG}'"
        '''
      }
    }

    stage('Gate for Promotion (main only)') {
      when { branch 'main' }
      steps {
        input message: "Promote ${APP_NAME}:${IMAGE_TAG} to Next Environment?", ok: 'Approve'
      }
    }
  } // stages

  post {
    success {
      echo "Build ${env.BUILD_TAG} succeeded for ${env.BRANCH_NAME}"
    }
    failure {
      echo "Build failed — see stage logs and archived reports."
    }
    always {
      script {
        echo "Cleaning workspace using deleteDir()"
        try {
          // deleteDir is built-in and available; safer than cleanWs() here
          deleteDir()
        } catch (err) {
          echo "deleteDir() failed: ${err}. If workspace plugin needed, consider installing Workspace Cleanup plugin."
        }
      }
    }
  }
}
