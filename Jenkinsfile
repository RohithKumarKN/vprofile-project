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

        stage('Quality Gate') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
