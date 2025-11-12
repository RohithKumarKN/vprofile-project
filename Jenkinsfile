pipeline {
    agent any

    tools {
        maven "MAVEN3.9"
        jdk   "JDK17"
    }

    environment {
        SNAP_REPO      = 'vprofile-snapshot'
        NEXUS_USER     = 'admin'
        NEXUS_PASS     = 'admin123'
        RELEASE_REPO   = 'vprofile-release'
        CENTRAL_REPO   = 'vpro-maven-central'
        NEXUSIP        = '172.31.47.37'
        NEXUSPORT      = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN    = 'nexuslogin'
        SONARSERVER    = 'sonarserver'
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
                sh 'mvn clean package -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/*.war', fingerprint: true
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('Sonar Analysis') {
            steps {
                withSonarQubeEnv("${SONARSERVER}") {
                    sh "mvn verify sonar:sonar -DskipTests -Dsonar.projectVersion=${env.BUILD_VERSION}"
                }
            }
        }

        stage('UploadArtifact') {
            steps {
                script {
                    if (!fileExists('target/vprofile-v2.war')) {
                        error "Artifact target/vprofile-v2.war not found."
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

        stage('Ansible Deploy to staging') {
            steps {
                ansiblePlaybook([
                    inventory   : 'ansible/stage.inventory',
                    playbook    : 'ansible/site.yml',
                    installation: 'ansible',
                    colorized   : true,
                    credentialsId: 'applogin',
                    disableHostKeyChecking: true,
                    extraVars   : [
                        USER: "admin",
                        PASS: "${NEXUS_PASS}",
                        nexusip: "${NEXUSIP}",
                        reponame: "${RELEASE_REPO}",
                        groupid: "QA",
                        artifactid: "vproapp",
                        build_version: "${env.BUILD_VERSION}"
                    ]
                ])
            }
        }
    }
}