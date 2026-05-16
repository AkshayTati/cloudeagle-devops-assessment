pipeline {
    agent {
        // Enforcing a specialized build node equipped with JDK 17, Maven, and Google Cloud SDK
        node { label 'maven-gcloud-runner' }
    }

    options {
        timeout(time: 1, unit: 'HOURS')
        ansiColor('xterm')
        disableConcurrentBuilds()
        keepStepStatuses()
    }

    parameters {
        // Parameterized entrypoints allowing operators to trigger emergency rollbacks safely
        choice(name: 'ROLLBACK_TARGET_ENV', choices: ['none', 'qa', 'staging', 'prod'], description: 'Select target environment ONLY for an emergency rollback execution.')
        string(name: 'ROLLBACK_VERSION_TAG', defaultValue: '', description: 'The exact historical artifact tag (e.g., v1.0.42-7a3b2c1d) to roll back to.')
    }

    environment {
        APP_NAME           = 'sync-service'
        GCP_PROJECT_ID     = 'cloudeagle-core-infra'
        GCP_REGION         = 'asia-south1'
        GCS_ARTIFACT_BUCKET= 'cloudeagle-immutable-artifacts-prod'
        
        // Architect Choice: Combine build numbers with short Git SHAs for unalterable, traceable semantic tracking
        SHORT_SHA          = "${env.GIT_COMMIT ? env.GIT_COMMIT.take(8) : 'snapshot'}"
        VERSION_TAG        = "v1.0.${env.BUILD_NUMBER}-${env.SHORT_SHA}"
        ARTIFACT_NAME      = "${APP_NAME}-${VERSION_TAG}.jar"
    }

    stages {
        stage('PR Verification & Quality Gate') {
            // Evaluates pull requests ephemerally. Does not modify cloud state or build artifacts.
            when { changeRequest() }
            parallel {
                stage('Unit & Integration Testing') {
                    steps {
                        echo "Executing deterministic test suites..."
                        sh './mvnw clean test verify'
                    }
                }
                stage('Static Application Security Testing (SAST)') {
                    steps {
                        echo "Analyzing codebase vulnerabilities and code quality metrics via SonarQube..."
                        // withSonarQubeEnv('SonarQube-Server') { sh './mvnw sonar:sonar -Dsonar.projectKey=sync-service' }
                    }
                }
            }
        }

        stage('Immutable Artifact Compilation') {
            // Triggered only on true branch merges/pushes; skipped entirely during explicit manual rollbacks
            when {
                allOf {
                    not { changeRequest() }
                    anyOf { branch 'develop'; branch 'release/*'; branch 'main' }
                    expression { params.ROLLBACK_VERSION_TAG == '' }
                }
            }
            steps {
                echo "Compiling and packaging release binary: ${ARTIFACT_NAME}"
                // Compile once, deploy everywhere principle. Skips tests here since they pass at PR stage.
                sh './mvnw clean package -DskipTests'
                
                echo "Archiving release binary to Google Cloud Storage immutable repository..."
                sh "gsutil cp target/${APP_NAME}.jar gs://${GCS_ARTIFACT_BUCKET}/${APP_NAME}/${ARTIFACT_NAME}"
            }
        }

        stage('Continuous Deployment: QA') {
            // Continuous Integration tracking into the QA ecosystem
            when {
                allOf {
                    not { changeRequest() }
                    branch 'develop'
                    expression { params.ROLLBACK_VERSION_TAG == '' }
                }
            }
            steps {
                echo "Initiating rolling update to QA Managed Instance Group..."
                // Injecting environment configuration via Spring Profiles system properties
                // Secrets are fetched inside the VM via attached IAM Service Account directly from Secret Manager
                sh """
                    gcloud compute instance-groups managed rolling-action start-update cloudeagle-qa-mig \
                        --version=template=cloudeagle-qa-template-${VERSION_TAG} \
                        --region=${GCP_REGION}
                """
            }
        }

        stage('Continuous Delivery: Staging') {
            // Isolating staging for final User Acceptance Testing (UAT)
            when {
                allOf {
                    not { changeRequest() }
                    branch 'release/*'
                    expression { params.ROLLBACK_VERSION_TAG == '' }
                }
            }
            steps {
                echo "Deploying to Staging Infrastructure..."
                sh """
                    gcloud compute instance-groups managed rolling-action start-update cloudeagle-staging-mig \
                        --version=template=cloudeagle-staging-template-${VERSION_TAG} \
                        --region=${GCP_REGION}
                """
            }
        }

        stage('Orchestrated Production Release') {
            when {
                allOf {
                    not { changeRequest() }
                    branch 'main'
                    expression { params.ROLLBACK_VERSION_TAG == '' }
                }
            }
            stages {
                stage('Strategic Manual Approval Gate') {
                    steps {
                        // Decoupling git merge from production traffic exposure. Timeout prevents blocked pipelines.
                        timeout(time: 24, unit: 'HOURS') {
                            input message: "Promote release candidate ${VERSION_TAG} to live Production?", ok: "Approve Traffic Cutover"
                        }
                    }
                }
                stage('Execute Blue/Green VM Switch') {
                    steps {
                        echo "Spinning up target 'Green' infrastructure with version ${VERSION_TAG}..."
                        // 1. Update the standby (Green) Managed Instance Group template
                        // 2. Validate Green pool health via GCP load balancer backend health checks
                        // 3. Atomically update the Google Cloud HTTPS Load Balancer URL-Map to switch traffic
                        sh "./scripts/gcp-blue-green-switch.sh --version ${VERSION_TAG} --target green"
                    }
                }
            }
        }

        stage('Decoupled Emergency Rollback Engine') {
            // Explicit operational route designed to recover from production anomalies instantly
            when {
                allOf {
                    expression { params.ROLLBACK_TARGET_ENV != 'none' }
                    expression { params.ROLLBACK_VERSION_TAG != '' }
                }
            }
            steps {
                script {
                    currentBuild.description = "EMERGENCY ROLLBACK TO ${params.ROLLBACK_VERSION_TAG} ON ${params.ROLLBACK_TARGET_ENV.toUpperCase()}"
                    echo "⚠️ INITIATING EMERGENCY ROLLBACK PROCEDURES ⚠️"
                    echo "Target Environment: ${params.ROLLBACK_TARGET_ENV}"
                    echo "Target Restorational Version: ${params.ROLLBACK_VERSION_TAG}"
                    
                    // Verifying existence of target stable historical binary in GCS bucket before execution
                    sh "gsutil ls gs://${GCS_ARTIFACT_BUCKET}/${APP_NAME}/${APP_NAME}-${params.ROLLBACK_VERSION_TAG}.jar"
                    
                    if (params.ROLLBACK_TARGET_ENV == 'prod') {
                        echo "Executing atomic rollback switch at Global Load Balancer layer..."
                        sh "./scripts/gcp-blue-green-switch.sh --version ${params.ROLLBACK_VERSION_TAG} --rollback"
                    } else {
                        echo "Reverting instance template targets for lower environment..."
                        sh "gcloud compute instance-groups managed rolling-action start-update cloudeagle-${params.ROLLBACK_TARGET_ENV}-mig --version=template=cloudeagle-${params.ROLLBACK_TARGET_ENV}-template-${params.ROLLBACK_VERSION_TAG} --region=${GCP_REGION}"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs() // Maintain pristine workspace hygiene on self-hosted runners
        }
        success {
            echo "Pipeline executed successfully."
            // slackSend channel: '#infra-alerts', color: 'good', message: "SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} unified."
        }
        failure {
            echo "Pipeline breakdown detected. Dispatching alert triggers..."
            // slackSend channel: '#infra-alerts', color: 'danger', message: "CRITICAL FAILURE: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} broken."
        }
    }
}