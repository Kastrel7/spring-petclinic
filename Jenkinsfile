pipeline {
    agent any

    tools {
        maven 'Maven-3.9'
        jdk   'JDK-17'
    }

    environment {
        SONAR_SERVER   = 'SonarQube'
        SONAR_TOKEN    = credentials('sonar-token')
        GITHUB_CREDS   = 'github-credentials'
        DEV_PORT       = '8081'
        STAGING_PORT   = '8082'
        DEV_DIR        = '/tmp/petclinic-dev'
        STAGING_DIR    = '/tmp/petclinic-staging'
        DOCKERHUB_REPO = 'kastrel7/spring-petclinic'
        HOST_IP        = '192.168.178.141'
        EC2_IP         = '16.171.21.39'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 60, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Install Dependencies') {
            steps {
                sh '''#!/bin/bash
                    apt-get update -qq
                    which docker || apt-get install -y -qq docker.io
                    which ansible || apt-get install -y -qq ansible
                    ansible-galaxy collection install community.docker
                '''
            }
        }

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

        stage('Build') {
            steps {
                sh 'mvn -B clean compile'
            }
        }

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

        stage('Package') {
            steps {
                sh 'mvn -B package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

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

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                // script {
                //     docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
                //         def image = docker.build("kastrel7/spring-petclinic:${BUILD_NUMBER}")
                //         image.push()
                //         image.push('latest')
                //     }
                // }

                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    ansiblePlaybook(
                        playbook: 'ansible/build-and-push.yml',
                        inventory: 'ansible/inventory.ini',
                        extraVars: [
                            docker_user: "${DOCKER_USER}",
                            docker_pass: "${DOCKER_PASS}",
                            build_number: "${BUILD_NUMBER}"
                        ]
                    )
                }
            }
        }

        stage('Deploy to Dev') {
            steps {
                // sh '''#!/bin/bash
                //     docker stop petclinic-dev || true
                //     docker rm petclinic-dev || true
                //     docker run -d \
                //         --name petclinic-dev \
                //         -p 8081:8080 \
                //         kastrel7/spring-petclinic:latest
                //     echo "Dev container started"
                // '''

                ansiblePlaybook(
                    playbook: 'ansible/deploy-to-dev.yml',
                    inventory: 'ansible/inventory.ini'
                )
            }
        }

        stage('Smoke Test (Dev)') {
            steps {
                // sh """
                //     echo "Waiting for app to start on port ${DEV_PORT}..."

                //     for i in \$(seq 1 12); do
                //         STATUS=\$(curl -s -o /dev/null -w "%{http_code}" \
                //             http://${HOST_IP}:${DEV_PORT}/actuator/health || echo "000")

                //         if [ "\$STATUS" = "200" ]; then
                //             echo "Smoke test PASSED - App is UP on dev (HTTP \$STATUS)"
                //             curl -s http://${HOST_IP}:${DEV_PORT}/actuator/health
                //             exit 0
                //         fi

                //         echo "Attempt \$i/12 - HTTP \$STATUS - retrying in 5s..."
                //         sleep 5
                //     done

                //     echo "SMOKE TEST FAILED - App did not start within 60 seconds"
                //     cat ${DEV_DIR}/app.log
                //     exit 1
                // """

                ansiblePlaybook(
                    playbook: 'ansible/smoke-test.yml',
                    inventory: 'ansible/inventory.ini',
                    extraVars: [url: "http://${HOST_IP}:${DEV_PORT}/actuator/health"]
                )
            }
        }

        stage('Approval: Promote to Staging?') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'Dev smoke tests passed. Promote to Staging?',
                          ok: 'Yes, promote to Staging',
                          submitter: 'admin'
                }
            }
        }

        // stage('Deploy to Staging') {
        //     steps {
        //         // sh '''#!/bin/bash
        //         //     docker stop petclinic-staging || true
        //         //     docker rm petclinic-staging || true
        //         //     docker run -d \
        //         //         --name petclinic-staging \
        //         //         -p 8082:8080 \
        //         //         kastrel7/spring-petclinic:latest
        //         //     echo "Staging container started"
        //         // '''

        //         ansiblePlaybook(
        //             playbook: 'ansible/deploy-to-staging.yml',
        //             inventory: 'ansible/inventory.ini'
        //         )
        //     }
        // }

        // stage('Smoke Test (Staging)') {
        //     steps {
        //         // sh """
        //         //     echo "Waiting for app to start on staging port ${STAGING_PORT}..."

        //         //     for i in \$(seq 1 12); do
        //         //         STATUS=\$(curl -s -o /dev/null -w "%{http_code}" \
        //         //             http://${HOST_IP}:${STAGING_PORT}/actuator/health || echo "000")

        //         //         if [ "\$STATUS" = "200" ]; then
        //         //             echo "Smoke test PASSED - App is UP on staging (HTTP \$STATUS)"
        //         //             curl -s http://${HOST_IP}:${STAGING_PORT}/actuator/health
        //         //             exit 0
        //         //         fi

        //         //         echo "Attempt \$i/12 - HTTP \$STATUS - retrying in 5s..."
        //         //         sleep 5
        //         //     done

        //         //     echo "SMOKE TEST FAILED - Staging app did not start within 60 seconds"
        //         //     cat ${STAGING_DIR}/app.log
        //         //     exit 1
        //         // """

        //         ansiblePlaybook(
        //             playbook: 'ansible/smoke-test.yml',
        //             inventory: 'ansible/inventory.ini',
        //             extraVars: [url: "http://${HOST_IP}:${STAGING_PORT}/actuator/health"]
        //         )
        //     }
        // }

        stage('Deploy to Staging (EC2)') {
            steps {
                ansiblePlaybook(
                    playbook: 'ansible/provision-ec2.yml',
                    inventory: 'ansible/inventory.ini'
                )
            }
        }

        stage('Smoke Test (EC2)') {
            steps {
                ansiblePlaybook(
                    playbook: 'ansible/smoke-test.yml',
                    inventory: 'ansible/inventory.ini',
                    extraVars: [url: "http://${EC2_IP}:8080/actuator/health"]
                )
            }
        }
    }

    post {
        success {
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                sh """
                    curl -s -X POST -H 'Content-type: application/json' \
                    --data '{"text":"BUILD SUCCESSFUL :white_check_mark: - ${env.JOB_NAME} #${env.BUILD_NUMBER} - Deployed to EC2"}' \
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