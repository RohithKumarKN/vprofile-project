// Optional Slack color map
def COLOR_MAP = [
  'SUCCESS':  'good',
  'FAILURE':  'danger',
  'UNSTABLE': '#FFCC00',
  'ABORTED':  '#AAAAAA',
  'NOT_BUILT':'#888888'
]

pipeline {
    agent any

    tools {
        maven "MAVEN3.9"
        jdk   "JDK21"
    }

    environment {
        SNAP_REPO      = 'vprofile-snapshot'
        NEXUS_USER     = 'admin'
        NEXUS_PASS     = 'admin'
        RELEASE_REPO   = 'vprofile-release'
        CENTRAL_REPO   = 'vpro-maven-central'
        NEXUSIP        = '172.31.39.58'
        NEXUSPORT      = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN    = 'nexuslogin'
        SONARSERVER    = 'sonarserver'
    }

    options { timestamps() }

    stages {
        stage('Build') {
            steps {
                sh 'mvn -s settings.xml -DskipTests install'
            }
            post {
                success {
                    echo "Now Archiving."
                    archiveArtifacts artifacts: '**/*.war', fingerprint: true
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -s settings.xml test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn -s settings.xml checkstyle:checkstyle'
            }
        }

        stage('Sonar Analysis') {
            steps {
                withSonarQubeEnv("${SONARSERVER}") {
                    sh "mvn -s settings.xml -DskipTests -Dsonar.projectVersion=${env.BUILD_VERSION} verify sonar:sonar"
                }
            }
        }

        stage('UploadArtifact') {
            steps {
                script {
                    if (!fileExists('target/vprofile-v2.war')) {
                        error "Artifact target/vprofile-v2.war not found. Check your packaging step or artifact name."
                    }
                }
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                    groupId: 'QA',
                    version: "${env.BUILD_VERSION}",
                    repository: "${RELEASE_REPO}",
                    credentialsId: "${NEXUS_LOGIN}",
                    artifacts: [[
                        artifactId: 'vproapp',
                        classifier: '',
                        file: 'target/vprofile-v2.war',
                        type: 'war'
                    ]]
                )
            }
        }
    }

    post {
        always {
            echo "Slack Notifications."
            script {
                def color = COLOR_MAP.get(currentBuild.currentResult, '#439FE0')
                slackSend(
                    channel: '#jenkinscicd',
                    color: color,
                    message: "*${currentBuild.currentResult ?: 'UNKNOWN'}*: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\nMore info: ${env.BUILD_URL}"
                )
            }
            echo "Pipeline finished: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}





