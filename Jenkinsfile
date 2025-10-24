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
        jdk   "JDK17"
    }

    

    environment {
        SNAP_REPO      = 'vprofile-snapshot'
        NEXUS_USER     = 'admin'
        NEXUS_PASS     = 'admin'
        RELEASE_REPO   = 'vprofile-release'
        CENTRAL_REPO   = 'vpro-maven-central'
        NEXUSIP        = '172.31.47.226'
        NEXUSPORT      = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN    = 'nexuslogin'
        SONARSERVER    = 'sonarserver'   // Must match Manage Jenkins > System > SonarQube servers (Name)
    }

    options { timestamps() }

    stages {
        stage('Prepare Version') {
            steps {
                script {
                    def ts = new Date().format("yyyyMMdd-HHmmss", TimeZone.getTimeZone('UTC'))
                    env.BUILD_VERSION = "${env.BUILD_NUMBER}-${ts}"
                    echo "BUILD_VERSION = ${env.BUILD_VERSION}"
                }
            }
        }

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
                // withSonarQubeEnv injects server URL/token for Maven sonar:sonar
                withSonarQubeEnv("${SONARSERVER}") {
                    sh "mvn -s settings.xml -DskipTests -Dsonar.projectVersion=${env.BUILD_VERSION} verify sonar:sonar"
                }
            }
        }

        // If you later re-enable it, ensure SonarQube webhook points to /sonarqube-webhook/
        // stage('Quality Gate') {
        //     steps {
        //         timeout(time: 30, unit: 'MINUTES') {
        //             waitForQualityGate abortPipeline: true
        //         }
        //     }
        // }

        stage('UploadArtifact') {
            steps {
                script {
                    // Sanity check: ensure artifact exists before uploading
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

    // ✅ Only ONE top-level post block
    post {
        always {
            echo "Slack Notifications."
            script {
                // If you don't use Slack, remove this whole script block
                def color = COLOR_MAP.get(currentBuild.currentResult, '#439FE0') // default Slack blue
                slackSend(
                    channel: '#jenkinscicd',
                    color: color,
                    message: "*${currentBuild.currentResult ?: 'UNKNOWN'}*: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\nMore info: ${env.BUILD_URL}"
                )
            }
            echo "Pipeline finished: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        // Optional clean-up
        // success { deleteDir() }
        // failure { deleteDir() }
    }
}