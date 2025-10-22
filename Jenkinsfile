pipeline {
    agent any

    tools {
        maven "MAVEN3.9"   // Make sure these tool names match Manage Jenkins > Global Tool Configuration
        jdk "JDK17"
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
    }

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
                failure {
                    echo "Build failed — not archiving."
                }
                always {
                    echo "Build stage done."
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -s settings.xml test'
            }
            post {
                always {
                    // Publish JUnit test results if present
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            // deleteDir() // enable if you want workspace cleanup
        }
    }
}