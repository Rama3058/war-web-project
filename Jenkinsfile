pipeline {
    agent any

    environment {
        // ---------- SonarQube ----------
        SONAR_HOST_URL = "http://13.126.135.101:9000"
        SONAR_CREDENTIAL_ID = "sonar_creds"

        // ---------- Nexus ----------
        NEXUS_URL = "15.207.55.128:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
    }

    tools {
        maven "maven"
    }

    stages {

        stage('Build WAR') {
            steps {
                echo "🔨 Building WAR..."
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.war', allowEmptyArchive: false
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "🔍 Running SonarQube analysis..."

                withCredentials([
                    usernamePassword(
                        credentialsId: 'sonar_creds',
                        usernameVariable: 'SONAR_USER',
                        passwordVariable: 'SONAR_PASS'
                    )
                ]) {
                    sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=wwp \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.login=${SONAR_USER} \
                        -Dsonar.password=${SONAR_PASS} \
                        -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage('Extract Version') {
            steps {
                script {
                    env.ART_VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()
                    echo "📦 Version: ${ART_VERSION}"
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

                    echo "🚀 Uploading WAR to Nexus..."

                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUS_URL}",
                        groupId: 'koddas.web.war',
                        artifactId: 'wwp',
                        version: "${ART_VERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [
                            [artifactId: 'wwp', file: warFile, type: 'war']
                        ]
                    )
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully"
            echo "🔗 SonarQube: http://13.126.135.101:9000"
            echo "📦 Nexus: http://15.207.55.128:8081"
        }
        failure {
            echo "❌ Pipeline failed — check logs carefully"
        }
    }
}
