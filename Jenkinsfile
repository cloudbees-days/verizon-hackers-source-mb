// Jenkinsfile (fixed) — Multibranch-ready, with real JUnit XML unit tests
pipeline {
  agent { label 'linux && docker' }  // keep your original agent

  options {
    // Removed timestamps() and ansiColor() because they caused invalid-option errors in your environment
    durabilityHint('MAX_SURVIVABILITY')
    buildDiscarder(logRotator(numToKeepStr: '30'))
    skipDefaultCheckout(true)
    timeout(time: 60, unit: 'MINUTES')
  }

  environment {
    APP_NAME        = 'sample-app'
    DOCKER_REGISTRY = 'registry.example.com'
    DOCKER_ORG      = 'verizon-demo'
    IMAGE_TAG       = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    SIGNING_KEY_ID  = credentials('codesign-key-id')       // optional
    ARTIFACTORY_CREDS = credentials('artifact-repo-creds') // optional (username/password => ARTIFACTORY_CREDS_USR/_PSW)
    RUN_DAST        = "${env.BRANCH_NAME == 'main' ? 'true' : 'false'}"
  }

  parameters {
    booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip unit/integration tests')
    booleanParam(name: 'SKIP_SAST', defaultValue: false, description: 'Skip SAST (not recommended)')
    booleanParam(name: 'SKIP_DAST', defaultValue: false, description: 'Skip DAST on main')
    choice(name: 'BUILD_KIND', choices: ['container', 'binary'], description: 'Build container image or non-container binary/package')
  }

  // NOTE: removed empty triggers {} block — Multibranch typically uses webhooks.

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git fetch --prune --unshallow || true'   // optional, safe
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
              # npm ci && npm run lint
            '''
          }
        }

        stage('Unit Tests') {
          when { expression { !params.SKIP_TESTS } }
          steps {
            // Create real JUnit XMLs so jenkins junit step shows details and system-out
            sh '''
              mkdir -p reports/junit

              # Test 1 (pass)
              cat > reports/junit/TEST-add.xml <<'XML'
              <?xml version="1.0" encoding="UTF-8"?>
              <testsuite tests="1" failures="0" name="add-suite">
                <testcase classname="sample.calculator" name="test_add" time="0.01">
                  <system-out>input=(3,4); expected=7; actual=7</system-out>
                </testcase>
              </testsuite>
              XML

              # Test 2 (pass)
              cat > reports/junit/TEST-mul.xml <<'XML'
              <?xml version="1.0" encoding="UTF-8"?>
              <testsuite tests="1" failures="0" name="mul-suite">
                <testcase classname="sample.calculator" name="test_mul" time="0.02">
                  <system-out>input=(3,4); expected=12; actual=12</system-out>
                </testcase>
              </testsuite>
              XML

              # Test 3 (failing - demonstrates failure metadata)
              cat > reports/junit/TEST-fail.xml <<'XML'
              <?xml version="1.0" encoding="UTF-8"?>
              <testsuite tests="1" failures="1" name="fail-suite">
                <testcase classname="sample.calculator" name="test_fail" time="0.005">
                  <failure message="expected 5 but got 4">AssertionError</failure>
                  <system-out>input=(2,2); expected=5; actual=4</system-out>
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
        } // Unit Tests

        stage('SAST') {
          when {
            allOf {
              expression { !params.SKIP_SAST }
              not { changeRequest() } // optionally skip on PRs
            }
          }
          steps {
            sh '''
              echo "Run SAST here (placeholder). Examples:"
              echo "- Semgrep: semgrep ci --json > reports/semgrep.json || true"
              echo "- SonarQube: mvn sonar:sonar (with Sonar env)"
              echo "- Bandit/Trivy FS/etc."
            '''
          }
          post {
            always {
              // Publish reports or convert to SARIF for Unify ingestion
              archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**'
            }
          }
        }
      }
    } // end parallel

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
          if (params.BUILD_KIND == 'container') {
            sh '''
              echo "Sign image (placeholder). Example with cosign:"
              # cosign sign --key env://COSIGN_KEY ${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG} || true
            '''
          } else {
            sh '''
              echo "Sign binary (placeholder). Example:"
              # gpg --batch --yes --detach-sign --armor -u "$SIGNING_KEY_ID" dist/app-binary || true
            '''
            archiveArtifacts artifacts: 'dist/**', fingerprint: true
          }
        }
      }
    }

    stage('Push Artifact/Image') {
      when { not { changeRequest() } } // typically skip for PRs
      steps {
        script {
          if (params.BUILD_KIND == 'container') {
            sh """
              echo "Login & push image (mock)"
              echo "${ARTIFACTORY_CREDS_PSW}" | docker login ${DOCKER_REGISTRY} -u "${ARTIFACTORY_CREDS_USR}" --password-stdin || true
              # docker push ${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG}
            """
          } else {
            sh '''
              echo "Upload binary to artifact repo (placeholder). Examples:"
              echo "- curl to Artifactory/Nexus/S3 with credentials"
            '''
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
      post {
        always {
          archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**'
        }
      }
    }

    stage('Publish Build Metadata (for Unify)') {
      steps {
        sh '''
          echo "Emit SARIF / JUnit / SBOM / signature metadata for Unify ingestion"
          # syft packages dir:. -o cyclonedx-json > reports/sbom.json || true
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
      // cleanup - runs on the agent because top-level agent is declared
      cleanWs deleteDirs: true, notFailBuild: true
    }
  }
}
