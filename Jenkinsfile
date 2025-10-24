// Jenkinsfile - Multibranch-ready, original stages kept.
// Fixes: removed invalid options (timestamps/ansiColor) and empty triggers,
// ensures post cleanup runs inside a node context, adds extra unit tests with system-out.

pipeline {
  agent { label 'linux && docker' }

  options {
    // keep durability and build discarder
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
    SIGNING_KEY_ID  = credentials('codesign-key-id')       // optional: replace or remove
    ARTIFACTORY_CREDS = credentials('artifact-repo-creds') // optional: replace or remove
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
        sh 'echo "BRANCH_NAME=${BRANCH_NAME:-$(git rev-parse --abbrev-ref HEAD || echo HEAD)}"'
        sh 'mkdir -p reports junit dist'
      }
    }

    stage('Prepare / Tooling') {
      steps {
        sh '''
          echo "Prepare tools - minimal checks"
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

                # Test 3 - greet
                cat > reports/junit/TEST-greet.xml <<'XML'
                <?xml version="1.0" encoding="UTF-8"?>
                <testsuite tests="1" failures="0" name="greet-suite">
                  <testcase classname="sample.greet" name="test_greeting" time="0.005">
                    <system-out>input="Shozab"; expected="Hello Shozab"; actual="Hello Shozab"</system-out>
                  </testcase>
                </testsuite>
                XML

                # Test 4 - edge-case
                cat > reports/junit/TEST-edge.xml <<'XML'
                <?xml version="1.0" encoding="UTF-8"?>
                <testsuite tests="1" failures="0" name="edge-suite">
                  <testcase classname="sample.calc" name="test_edge" time="0.007">
                    <system-out>input=(0,-1); expected=-1; actual=-1</system-out>
                  </testcase>
                </testsuite>
                XML

                # Test 5 - failing example (to demonstrate failures)
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
          when { expression { !params.SKIP_SAST } }
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
      }
    } // end parallel

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
      when { not { changeRequest() } }
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
        script { timeout(time: 30, unit: 'MINUTES') { input message: "Promote ${APP_NAME}:${IMAGE_TAG} to the next environment?", ok: 'Approve' } }
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
          sh '''
            echo "Prod Stage: deploy approved artifact to Prod (mock)"
            mkdir -p reports/prod && echo "prod-deploy:ok" > reports/prod/out.txt
          '''
        }
        archiveArtifacts artifacts: 'reports/prod/**', allowEmptyArchive: true
      }
    }
  } // end stages

  post {
    success { echo "Build ${env.BUILD_TAG} succeeded for ${env.BRANCH_NAME}" }
    failure { echo "Build ${env.BUILD_TAG} failed for ${env.BRANCH_NAME}" }
    always {
      // Ensure cleanup runs inside a node context to avoid 'agent none' errors
      script {
        if (env.NODE_NAME) {
          echo "Cleaning workspace on node ${env.NODE_NAME}..."
          try { cleanWs() } catch (err) { echo "cleanWs() failed: ${err}"; deleteDir() }
        } else {
          node {
            echo "Post had no node; performing cleanup inside node()"
            try { cleanWs() } catch (err) { echo "cleanWs() failed in fallback: ${err}"; deleteDir() }
          }
        }
      }
    }
  }
}
