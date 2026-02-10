pipeline {
    agent any
    
    tools {
        maven 'MAVEN3'
        jdk 'OracleJDK8'
    }
    
    environment {
        // Nexus Configuration
        SNAP_REPO = 'vprofile-snapshot'
        NEXUS_USER = 'admin'
        NEXUS_PASS = 'Uma@1827'
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUS_IP = '172.31.46.191'
        NEXUS_PORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN = 'nexuslogin'
        
        // SonarQube Configuration
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner4'
        
        // Artifact versioning
        ARTIFACT_VERSION = "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                script {
                    echo "========== Starting Maven Build =========="
                    sh 'mvn -s settings.xml -DskipTests clean install'
                }
            }
            post {
                success {
                    echo 'Build successful! Archiving artifacts...'
                    archiveArtifacts artifacts: '**/target/*.war', fingerprint: true
                }
                failure {
                    echo 'Build failed. Check Maven logs for details.'
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                script {
                    echo "========== Running Unit Tests =========="
                    sh 'mvn -s settings.xml test'
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Integration Tests') {
            steps {
                script {
                    echo "========== Running Integration Tests =========="
                    sh 'mvn -s settings.xml verify -DskipUnitTests'
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/failsafe-reports/*.xml'
                }
            }
        }
        
        stage('Checkstyle Analysis') {
            steps {
                script {
                    echo "========== Running Checkstyle Analysis =========="
                    sh 'mvn -s settings.xml checkstyle:checkstyle'
                }
            }
            post {
                always {
                    recordIssues enabledForFailure: true, tool: checkStyle(pattern: '**/target/checkstyle-result.xml')
                }
            }
        }
        
        stage('Code Coverage') {
            steps {
                script {
                    echo "========== Generating Code Coverage Report =========="
                    sh 'mvn -s settings.xml jacoco:report'
                }
            }
            post {
                always {
                    jacoco execPattern: '**/target/jacoco.exec'
                }
            }
        }
        
        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
                script {
                    echo "========== Running SonarQube Analysis =========="
                    withSonarQubeEnv("${SONARSERVER}") {
                        sh """${scannerHome}/bin/sonar-scanner \
                           -Dsonar.projectKey=vprofile \
                           -Dsonar.projectName=vprofile-repo \
                           -Dsonar.projectVersion=1.0 \
                           -Dsonar.sources=src/ \
                           -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                           -Dsonar.junit.reportsPath=target/surefire-reports/ \
                           -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                           -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml"""
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    echo "========== Waiting for Quality Gate Result =========="
                    timeout(time: 10, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        } else {
                            echo "Quality Gate Passed!"
                        }
                    }
                }
            }
        }
        
        stage('Upload Artifact to Nexus') {
            steps {
                script {
                    echo "========== Uploading Artifact to Nexus =========="
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUS_IP}:${NEXUS_PORT}",
                        groupId: 'QA',
                        version: "${ARTIFACT_VERSION}",
                        repository: "${RELEASE_REPO}",
                        credentialsId: "${NEXUS_LOGIN}",
                        artifacts: [
                            [
                                artifactId: 'vproapp',
                                classifier: '',
                                file: 'target/vprofile-v2.war',
                                type: 'war'
                            ]
                        ]
                    )
                }
            }
            post {
                success {
                    echo "Artifact successfully uploaded to Nexus repository: ${RELEASE_REPO}"
                    echo "Artifact version: ${ARTIFACT_VERSION}"
                }
                failure {
                    echo "Failed to upload artifact to Nexus. Check credentials and connection."
                }
            }
        }
    }
    
    post {
        always {
            echo "========== Pipeline Execution Completed =========="
            cleanWs()
        }
        
        success {
            echo "========== Build SUCCESSFUL =========="
            slackSend(
                channel: '#jenkinscicd',
                color: 'good',
                message: """*SUCCESS* ✅
                |Job: ${env.JOB_NAME}
                |Build: #${env.BUILD_NUMBER}
                |Duration: ${currentBuild.durationString}
                |Version: ${ARTIFACT_VERSION}
                |More info: ${env.BUILD_URL}""".stripMargin()
            )
        }
        
        failure {
            echo "========== Build FAILED =========="
            slackSend(
                channel: '#jenkinscicd',
                color: 'danger',
                message: """*FAILURE* ❌
                |Job: ${env.JOB_NAME}
                |Build: #${env.BUILD_NUMBER}
                |Duration: ${currentBuild.durationString}
                |Failed at: ${currentBuild.currentResult}
                |More info: ${env.BUILD_URL}console""".stripMargin()
            )
        }
        
        unstable {
            echo "========== Build UNSTABLE =========="
            slackSend(
                channel: '#jenkinscicd',
                color: 'warning',
                message: """*UNSTABLE* ⚠️
                |Job: ${env.JOB_NAME}
                |Build: #${env.BUILD_NUMBER}
                |Duration: ${currentBuild.durationString}
                |More info: ${env.BUILD_URL}""".stripMargin()
            )
        }
    }
}
