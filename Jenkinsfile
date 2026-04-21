pipeline {
    agent any

    tools {
        maven 'Maven-3.9'
        jdk   'JDK-17'
    }

    environment {
        SONAR_SERVER  = 'SonarQube'
        SONAR_TOKEN   = credentials('sonar-token')
        GITHUB_CREDS  = 'github-credentials'
        // Port the app will run on inside the Jenkins container for Dev
        DEV_PORT      = '8081'
        // Port for Staging
        STAGING_PORT  = '8082'
        // Path where the JAR will be copied for each environment
        DEV_DIR       = '/tmp/petclinic-dev'
        STAGING_DIR   = '/tmp/petclinic-staging'
        DOCKERHUB_REPO = 'kastrel7/spring-petclinic'
        HOST_IP       = '192.168.178.141'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Increased to 60 min to account for the manual approval wait time
        timeout(time: 60, unit: 'MINUTES')
        timestamps()
    }

    stages {

        // ── Stage 1: Checkout ──────────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/Kastrel7/spring-petclinic.git',
                        credentialsId: "${GITHUB_CREDS}"
                    ]]
                ])
                echo "Checked out commit: ${GIT_COMMIT}"
            }
        }

        // ── Stage 2: Compile ───────────────────────────────────────────────────────
        stage('Build') {
            steps {
                sh 'mvn -B clean compile'
            }
        }

        // ── Stage 3: Unit Tests + Coverage ────────────────────────────────────────
        stage('Unit Tests') {
            steps {
                sh 'mvn -B test -Dtest="!PostgresIntegrationTests"'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    jacoco(
                        execPattern:    '**/target/jacoco.exec',
                        classPattern:   '**/target/classes',
                        sourcePattern:  '**/src/main/java'
                    )
                }
            }
        }

        // ── Stage 4: Package ───────────────────────────────────────────────────────
        stage('Package') {
            steps {
                sh 'mvn -B package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        // ── Stage 5: SonarQube Analysis ───────────────────────────────────────────
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONAR_SERVER}") {
                    sh """
                        mvn -B sonar:sonar \
                            -Dsonar.projectKey=spring-petclinic \
                            -Dsonar.projectName='Spring PetClinic' \
                            -Dsonar.token=${SONAR_TOKEN} \
                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    """
                }
            }
        }

        // ── Stage 6: Quality Gate ─────────────────────────────────────────────────
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ── Stage 7: Build & Push Docker Image ───────────────────────────────
        stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''#!/bin/bash
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker build -t kastrel7/spring-petclinic:latest \
                                    -t kastrel7/spring-petclinic:${BUILD_NUMBER} .
                        docker push kastrel7/spring-petclinic:latest
                        docker push kastrel7/spring-petclinic:${BUILD_NUMBER}
                        docker logout
                    '''
                }
            }
        }

        // ── Stage 8: Deploy to Dev ────────────────────────────────────────────────
        // Copies the JAR into a dev directory and starts it on DEV_PORT.
        // Any existing process on that port is killed first for a clean deploy.
        stage('Deploy to Dev') {
            steps {
                sh '''#!/bin/bash
                    docker stop petclinic-dev || true
                    docker rm petclinic-dev || true
                    docker run -d \
                        --name petclinic-dev \
                        -p 8081:8080 \
                        kastrel7/spring-petclinic:latest
                    echo "Dev container started"
                '''
            }
        }

        // ── Stage 9: Smoke Test (Dev) ─────────────────────────────────────────────
        // Polls the Spring Boot Actuator health endpoint every 5s for up to 60s.
        // Fails the build if the app does not report UP within that window.
        stage('Smoke Test (Dev)') {
            steps {
                sh """
                    echo "Waiting for app to start on port ${DEV_PORT}..."

                    for i in \$(seq 1 12); do
                        STATUS=\$(curl -s -o /dev/null -w "%{http_code}" \
                            http://${HOST_IP}:${DEV_PORT}/actuator/health || echo "000")

                        if [ "\$STATUS" = "200" ]; then
                            echo "Smoke test PASSED - App is UP on dev (HTTP \$STATUS)"
                            curl -s http://${HOST_IP}:${DEV_PORT}/actuator/health
                            exit 0
                        fi

                        echo "Attempt \$i/12 - HTTP \$STATUS - retrying in 5s..."
                        sleep 5
                    done

                    echo "SMOKE TEST FAILED - App did not start within 60 seconds"
                    cat ${DEV_DIR}/app.log
                    exit 1
                """
            }
        }

        // ── Stage 10: Manual Approval Gate ────────────────────────────────────────
        // Pauses the pipeline and waits for a human to approve promotion.
        // Auto-aborts after 10 minutes if no action is taken.
        stage('Approval: Promote to Staging?') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'Dev smoke tests passed. Promote to Staging?',
                          ok: 'Yes, promote to Staging',
                          submitter: 'admin'
                }
            }
        }

        // ── Stage 11: Deploy to Staging ───────────────────────────────────────────
        // Promotes the same JAR that passed Dev smoke tests to the Staging environment.
        stage('Deploy to Staging') {
            steps {
                sh '''#!/bin/bash
                    docker stop petclinic-staging || true
                    docker rm petclinic-staging || true
                    docker run -d \
                        --name petclinic-staging \
                        -p 8082:8080 \
                        kastrel7/spring-petclinic:latest
                    echo "Staging container started"
                '''
            }
        }

        // ── Stage 12: Smoke Test (Staging) ───────────────────────────────────────
        // Same health check pattern as Dev but against the staging port.
        stage('Smoke Test (Staging)') {
            steps {
                sh """
                    echo "Waiting for app to start on staging port ${STAGING_PORT}..."

                    for i in \$(seq 1 12); do
                        STATUS=\$(curl -s -o /dev/null -w "%{http_code}" \
                            http://${HOST_IP}:${STAGING_PORT}/actuator/health || echo "000")

                        if [ "\$STATUS" = "200" ]; then
                            echo "Smoke test PASSED - App is UP on staging (HTTP \$STATUS)"
                            curl -s http://${HOST_IP}:${STAGING_PORT}/actuator/health
                            exit 0
                        fi

                        echo "Attempt \$i/12 - HTTP \$STATUS - retrying in 5s..."
                        sleep 5
                    done

                    echo "SMOKE TEST FAILED - Staging app did not start within 60 seconds"
                    cat ${STAGING_DIR}/app.log
                    exit 1
                """
            }
        }

        // ── Stage 13: Ansible: Provision & Deploy ───────────────────────────────────────
        stage('Ansible: Provision & Deploy') {
            steps {
                sh '''#!/bin/bash
                    cd ansible
                    ansible-playbook -i inventory.ini setup-environment.yml
                    ansible-playbook -i inventory.ini deploy-container.yml
                '''
            }
        }
    }

    // ── Post-build Slack notifications ────────────────────────────────────────────
    post {
        success {
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                sh """
                    curl -s -X POST -H 'Content-type: application/json' \
                    --data '{"text":"BUILD SUCCESSFUL :white_check_mark: - ${env.JOB_NAME} #${env.BUILD_NUMBER} - Deployed to Staging"}' \
                    \$SLACK_WEBHOOK
                """
            }
        }
        failure {
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                sh """
                    curl -s -X POST -H 'Content-type: application/json' \
                    --data '{"text":"BUILD FAILED :x: - ${env.JOB_NAME} #${env.BUILD_NUMBER}"}' \
                    \$SLACK_WEBHOOK
                """
            }
        }
        aborted {
            // Fires when the manual approval gate times out or is rejected
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                sh """
                    curl -s -X POST -H 'Content-type: application/json' \
                    --data '{"text":"BUILD ABORTED :no_entry: - ${env.JOB_NAME} #${env.BUILD_NUMBER} - Staging promotion was not approved"}' \
                    \$SLACK_WEBHOOK
                """
            }
        }
    }
}