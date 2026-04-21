pipeline {
    agent any

    // ── Tool aliases must match names configured in Jenkins > Manage Jenkins > Tools ──
    tools {
        maven 'Maven-3.9'
        jdk   'JDK-17'
    }

    environment {
        // SonarQube server name as configured in Jenkins > Configure System
        SONAR_SERVER      = 'SonarQube'
        // The token credential ID you saved in Jenkins > Credentials
        SONAR_TOKEN       = credentials('sonar-token')
        // Slack channel to post notifications to
        SLACK_CHANNEL     = '#devops-petclinic'
        // Jenkins credential ID holding your GitHub personal access token
        GITHUB_CREDS      = 'github-credentials'
    }

    options {
        // Keep only the last 10 builds to save disk space
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Fail the build if it runs for more than 30 minutes
        timeout(time: 30, unit: 'MINUTES')
        // Add timestamps to all console log lines
        timestamps()
    }

    stages {

        // ── Stage 1: Pull the source code from GitHub ──────────────────────────────
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

        // ── Stage 2: Compile the Java source code ──────────────────────────────────
        stage('Build') {
            steps {
                // -B = batch mode (no progress bars), cleaner logs in Jenkins
                sh 'mvn -B clean compile'
            }
        }

        // ── Stage 3: Run JUnit tests and generate JaCoCo coverage report ───────────
        stage('Unit Tests') {
            steps {
                sh 'mvn -B test'
            }
            post {
                always {
                    // Publish JUnit test results so Jenkins shows pass/fail counts
                    junit '**/target/surefire-reports/*.xml'
                    // Publish JaCoCo coverage report as a Jenkins graph
                    jacoco(
                        execPattern:    '**/target/jacoco.exec',
                        classPattern:   '**/target/classes',
                        sourcePattern:  '**/src/main/java'
                    )
                }
            }
        }

        // ── Stage 4: Package the app into a runnable JAR ───────────────────────────
        stage('Package') {
            steps {
                // -DskipTests skips running tests again since we already ran them
                sh 'mvn -B package -DskipTests'
                // Archive the JAR so it's downloadable from the Jenkins build page
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        // ── Stage 5: Run SonarQube static analysis ─────────────────────────────────
        stage('SonarQube Analysis') {
            steps {
                // withSonarQubeEnv injects SONAR_HOST_URL and auth automatically
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

        // ── Stage 6: Wait for SonarQube to process results and check the gate ──────
        stage('Quality Gate') {
            steps {
                // Pauses the pipeline and polls SonarQube until analysis is complete
                // abortPipeline: true means the build fails if the gate fails
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }

    // ── Post-build actions: send Slack notification regardless of outcome ──────────
    post {
        success {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: 'good',
                message: """
                    *BUILD SUCCESSFUL* :white_check_mark:
                    *Job:* ${env.JOB_NAME}
                    *Build:* #${env.BUILD_NUMBER}
                    *Commit:* ${env.GIT_COMMIT?.take(7)}
                    *Duration:* ${currentBuild.durationString}
                    <${env.BUILD_URL}|View Build>
                """.stripIndent()
            )
        }
        failure {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: 'danger',
                message: """
                    *BUILD FAILED* :x:
                    *Job:* ${env.JOB_NAME}
                    *Build:* #${env.BUILD_NUMBER}
                    *Commit:* ${env.GIT_COMMIT?.take(7)}
                    *Stage Failed:* ${env.STAGE_NAME}
                    <${env.BUILD_URL}|View Build>
                """.stripIndent()
            )
        }
        unstable {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: 'warning',
                message: """
                    *BUILD UNSTABLE* :warning:
                    *Job:* ${env.JOB_NAME}
                    *Build:* #${env.BUILD_NUMBER}
                    *Commit:* ${env.GIT_COMMIT?.take(7)}
                    <${env.BUILD_URL}|View Build>
                """.stripIndent()
            )
        }
    }
}