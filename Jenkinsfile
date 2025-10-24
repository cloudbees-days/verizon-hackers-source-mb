// Jenkinsfile - Multibranch-ready; retains original stages but fixes post cleanup (node context).
pipeline {
  // Use your original agent label; keeps behavior consistent with previous file.
  agent { label 'linux && docker' }

  options {
    timestamps()
    ansiColor('xterm')
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
    SIGNING_KEY_ID  = credentials('codesign-key-id')       // optional
    ARTIFACTORY_CREDS = credentials('artifact-repo-creds') // optional
    RUN_DAST        = "${env.BRANCH_NAME == 'main' ? 'true' : 'false'}"
  }

  parameters {
    booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip unit/integration tests')
    booleanParam(name: 'SKIP_SAST', defaultValue: false, description: 'Skip SAST (not recommended)')
    booleanParam(name: 'SKIP_DAST', defaultValue: false, description: 'Skip DAST on main')
    choice(name: 'BUILD_KIND', choices: ['container', 'binary'], description: 'Build container image or non-container binary/package')
  }

  triggers {
    // Multibranch webhooks typically used; optional cron:
    // pollSCM('H/5 * * * *')
  }

  stages {
    stage('Checkout') {
      steps {
        // Let Multibranch populate scm
        checkout scm
        sh 'echo "BRANCH_NAME=${BRANCH_NAME:-$(git rev-parse --abbrev-ref HEAD || echo HEAD)}"'
        sh 'mkdir -p reports junit dist'
      }
    }

    stage('Prepare / Tooling') {
      steps {
        sh '''
          echo "Prepare tools (node/java/go) - minimal checks"
          java -version || true
          node -v || true
          go version || true
        '''
      }
    }

    stage('Static Analysis & Tests (parallel)') {
      parallel {
        stage('Mock Unit Tests') {
          steps {
            script {
              // Create several JUnit XML files with system-out output for visibility
              sh '''
                mkdir -p reports/junit

                # Test 1 - add
                cat > reports/junit/TEST-add.xml <<'XML'
                <?xml version="1.0" encoding="UTF-8"?>
                <testsuite tests="1" failures="0" name="add-suite">
                  <testcase classname="sample.calculator" name="test_add" time="0.01">
                    <system-out>input=(3,4); expected=7; actual=7</system-out>
                  </testcase>
                </testsuite>
                XML

                # Test 2 - mul
                cat > reports/junit/TEST-mul.xml <<'XML'
                <?xml version="1.0" encoding="UTF-8"?>
                <testsuite tests="1" failures="0" name="mul-suite">
                  <testcase classname="sample.calculator" name="test_mul" time="0.02">
                    <system-out>input=(3,4); expected=12; actual=12</system-out>
                  </testcase>
                </testsuite>
                XML

                # Test 3 - failing example (optional)
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
          }
          post {
            always {
              junit allowEmptyResults: false, testResults: 'reports/junit/*.xml'
              archiveArtifacts artifacts: 'reports/junit/*.xml', allowEmptyArchive: true
            }
          }
        } // Mock Unit Tests

        stage('Lint / Static Checks') {
          steps {
            sh '''
              mkdir -p reports/lint
              echo "Mock lint output: OK" > reports/lint/lint.txt
            '''
          }
          post { always { archiveArtifacts artifacts: 'reports/lint/**', allowEmptyArchive: true } }
        }

        stage('SAST (produce SARIF)') {
          when {
            expression { !params.SKIP_SAST }
          }
          steps {
            sh '''
              mkdir -p reports/sarif
              cat > reports/sarif/result.sarif <<'SARIF'
              { "version":"2.1.0", "runs":[ { "tool": { "driver": { "name":"mock-sast" } }, "results": [] } ] }
              SARIF
              echo "Mock SARIF created"
            '''
          }
          post { always { archiveArtifacts artifacts: 'reports/sarif/**', allowEmptyArchive: true } }
        }
      } // parallel
    } // Static Analysis & Tests

    stage('Build') {
      steps {
        script {
          if (params.BUILD_KIND == 'container') {
            sh '''
              echo "Simulating docker build (placeholder)."
              mkdir -p dist
              echo "container image placeholder" > dist/image.txt
            '''
          } else {
            sh '''
              echo "Building non-container binary/package (placeholder)."
              mkdir -p dist
              echo "binary content" > dist/${APP_NAME}
            '''
          }
        }
      }
      post { always { archiveArtifacts artifacts: 'dist/**', fingerprint: true, allowEmptyArchive: true } }
    }

    stage('Sign & Package') {
      steps {
        script {
          if (params.BUILD_KIND == 'container') {
            sh '''
              mkdir -p dist/signature
              echo "signed-image:${IMAGE_TAG}" > dist/signature/image.sig
            '''
          } else {
            sh '''
              mkdir -p dist/signature
              echo "signed-binary:${IMAGE_TAG}" > dist/signature/${APP_NAME}.sig
            '''
          }
        }
      }
      post { always { archiveArtifacts artifacts: 'dist/**', fingerprint: true, allowEmptyArchive: true } }
    }

    stage('Push Artifact/Image') {
      when { not { changeRequest() } } // skip on PRs
      steps {
        script {
          if (params.BUILD_KIND == 'container') {
            sh '''
              echo "Pushing container to registry (mock). Replace with docker login/push."
              # docker login ... && docker push ...
            '''
          } else {
            sh '''
              echo "Upload binary to artifact repo (mock). Replace with curl/ci plugin."
            '''
          }
        }
      }
      post { always { archiveArtifacts artifacts: 'dist/**', allowEmptyArchive: true } }
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
          echo "Running DAST (mock) against test endpoint - replace with ZAP / Arachni etc."
          mkdir -p reports/dast
          echo "dast: OK" > reports/dast/dast.txt
        '''
      }
      post { always { archiveArtifacts artifacts: 'reports/dast/**', allowEmptyArchive: true } }
    }

    stage('Publish Build Metadata') {
      steps {
        sh '''
          mkdir -p reports/metadata
          echo "{ \\"artifact\\": \\"${APP_NAME}\\", \\"tag\\": \\"${IMAGE_TAG}\\" }" > reports/metadata/build.json
        '''
        archiveArtifacts artifacts: 'reports/metadata/**', allowEmptyArchive: true
      }
    }

    stage('Deploy to Dev (optional)') {
      when { branch pattern: "feature/.*", comparator: "REGEXP" }
      steps {
        sh '''
          echo "Deploying to dev (mock). Replace with helm/oc/kubectl."
          mkdir -p reports/deploy && echo "dev-deploy:ok" > reports/deploy/out.txt
        '''
      }
      post { always { archiveArtifacts artifacts: 'reports/deploy/**', allowEmptyArchive: true } }
    }

    stage('Gate for Promotion (main only)') {
      when { branch 'main' }
      steps {
        script {
          // interactive promotion input
          timeout(time: 30, unit: 'MINUTES') {
            input message: "Promote ${APP_NAME}:${IMAGE_TAG} to the next environment?", ok: 'Approve'
          }
        }
      }
    }

    stage('Dev') {
      when { branch 'develop' }
      steps {
        sh '''
          echo "Dev Stage: deploy to dev cluster (mock)"
          mkdir -p reports/dev && echo "dev-deploy:ok" > reports/dev/out.txt
        '''
        archiveArtifacts artifacts: 'reports/dev/**', allowEmptyArchive: true
      }
    }

    stage('Test') {
      steps {
        sh '''
          echo "Test Stage: run integration tests and optional DAST (mock)"
          mkdir -p reports/test && echo "integration:ok" > reports/test/out.txt
        '''
        archiveArtifacts artifacts: 'reports/test/**', allowEmptyArchive: true
      }
    }

    stage('Prod') {
      when { branch 'main' }
      steps {
        script {
          // manual approval handled earlier; do final prod deploy (mock)
          sh '''
            echo "Prod Stage: deploy approved artifact to Prod (mock)"
            mkdir -p reports/prod && echo "prod-deploy:ok" > reports/prod/out.txt
          '''
        }
        archiveArtifacts artifacts: 'reports/prod/**', allowEmptyArchive: true
      }
    }
  } // stages

  post {
    success {
      echo "Build ${env.BUILD_TAG} succeeded for ${env.BRANCH_NAME}"
    }
    failure {
      echo "Build ${env.BUILD_TAG} failed for ${env.BRANCH_NAME}"
    }
    always {
      // Ensure cleanup runs inside a node context. If post executes without a node, we allocate one.
      script {
        if (env.NODE_NAME) {
          // We already have a node context; use cleanWs()
          echo "Cleaning workspace on node ${env.NODE_NAME}..."
          try {
            cleanWs()
          } catch (err) {
            echo "cleanWs() failed: ${err}"
            // fallback
            deleteDir()
          }
        } else {
          // Post ran with no node. Allocate a node to clean workspace to avoid 'agent none' error.
          node {
            echo "Post had no node; performing cleanup inside node()"
            try {
              cleanWs()
            } catch (err) {
              echo "cleanWs() in fallback node failed: ${err}; trying deleteDir()"
              deleteDir()
            }
          }
        }
      }
    }
  }
}
