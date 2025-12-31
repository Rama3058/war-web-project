pipeline {
    agent any

    environment {
        NEXUS_URL = "172.31.3.179:8081"          // Your Nexus server (private IP)
        NEXUS_REPOSITORY = "maven-releases"      // Nexus repository name
        NEXUS_CREDENTIAL_ID = "nexus_creds"      // Jenkins-stored credentials ID
    }

    tools {
        maven "maven"                             // Maven installation name in Jenkins
    }

    stages {

        stage('Build WAR') {
            steps {
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: '**/target/*.war'
            }
        }

        stage('Extract Version') {
            steps {
                script {
                    env.ART_VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()
                    echo "🔖 Artifact version: ${env.ART_VERSION}"
                }
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    def warFile = sh(
                        script: 'find target -name "*.war" -print -quit',
                        returnStdout: true
                    ).trim()

                    echo "📦 Uploading WAR: ${warFile} to Nexus..."

                    // Clean Nexus URL to avoid double 'http://'
                    def nexusBaseUrl = NEXUS_URL.replaceAll('^http://', '').replaceAll('/$', '')

                    nexusArtifactUploader(
                        nexusVersion: "nexus3",
                        protocol: "http",
                        nexusUrl: nexusBaseUrl,
                        groupId: "koddas.web.war",
                        artifactId: "wwp",
                        version: "${env.ART_VERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        type: "war",
                        file: warFile,
                        allowOverwrite: true
                    )

                    echo "✅ Successfully uploaded: http://${nexusBaseUrl}/repository/${NEXUS_REPOSITORY}/koddas/web/war/wwp/${env.ART_VERSION}/wwp-${env.ART_VERSION}.war"
                }
            }
        }

    }

    post {
        success {
            echo '🎉 Nexus upload pipeline completed successfully!'
        }
        failure {
            echo '❌ Nexus upload pipeline failed. Check logs.'
        }
    }
}
