// Jenkinsfile (Declarative) — Multibranch-ready
// Works with PRs, feature branches, tags, and main. Includes gated release steps on main.
// Safe defaults + clear extension points for CBCI: scanners, artifact signing, uploads, etc.

pipeline {
  agent any
  options {
-   timestamps()
-   ansiColor('xterm')
    durabilityHint('MAX_SURVIVABILITY')
    buildDiscarder(logRotator(numToKeepStr: '30'))
    skipDefaultCheckout(true)
    timeout(time: 60, unit: 'MINUTES')
  }

  environment {
    APP_NAME        = 'sample-app'
    // Change registry/org to yours if building containers
    DOCKER_REGISTRY = 'registry.example.com'
    DOCKER_ORG      = 'verizon-demo'
    IMAGE_TAG       = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    // If you sign artifacts/images, wire your credentials/keys here
    SIGNING_KEY_ID  = credentials('codesign-key-id')       // optional
    ARTIFACTORY_CREDS = credentials('artifact-repo-creds') // optional
    // Useful flags
    RUN_DAST        = "${env.BRANCH_NAME == 'main' ? 'true' : 'false'}"
  }

  parameters {
    booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip unit/integration tests')
    booleanParam(name: 'SKIP_SAST', defaultValue: false, description: 'Skip SAST (not recommended)')
    booleanParam(name: 'SKIP_DAST', defaultValue: false, description: 'Skip DAST on main')
    choice(name: 'BUILD_KIND', choices: ['container', 'binary'], description: 'Build container image or non-container binary/package')
  }

- triggers {
-   // Webhook-based triggers come from Multibranch automatically; a cron is optional
-   // pollSCM('H/5 * * * *')
- }
+ // No triggers block needed for Multibranch; webhooks/PR events handle this.

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        // optionally fetch full history for versioning/changelog
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
              # npm ci && npm run lint
            '''
          }
        }
        stage('Unit Tests') {
          when { expression { !params.SKIP_TESTS } }
          steps {
            sh '''
              echo "Run unit tests"
              # npm test -- --json --outputFile=reports/jest.json
              # mvn -B -DskipITs test
            '''
          }
          post {
            always {
              junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml, **/junit*.xml'
              archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**'
            }
          }
        }
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
    }

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
              # mvn -B -DskipTests package
              # or: gradle build; or: go build ./...
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
              echo "Login & push image"
              echo "${ARTIFACTORY_CREDS_PSW}" | docker login ${DOCKER_REGISTRY} -u "${ARTIFACTORY_CREDS_USR}" --password-stdin
              docker push ${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}:${IMAGE_TAG}
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
          # Example SBOM:
          # syft packages dir:. -o cyclonedx-json > reports/sbom.json || true
        '''
        archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**'
      }
    }

    // Release stages are typically orchestrated by Unify (Ed.2+).
    // If you still want a Jenkins deploy for lower envs, add it here guarded by branch.
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
      cleanWs deleteDirs: true, notFailBuild: true
    }
  }
}
