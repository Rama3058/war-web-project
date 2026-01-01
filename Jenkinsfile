pipeline {
    agent any

    environment {
        // -------- Git --------
        GIT_REPO_URL = "https://github.com/mchetan-aradhya/war-web-project.git"

        // -------- SonarQube --------
        SONAR_HOST_URL = "http://13.126.135.101:9000"
        SONAR_CREDENTIAL_ID = "sonar_creds"   // Jenkins Secret Text (Sonar Token)

        // -------- Nexus --------
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
                echo "🔨 Building WAR file..."
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.war', allowEmptyArchive: false
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "🔍 Running SonarQube analysis..."

                withCredentials([
                    string(credentialsId: "${SONAR_CREDENTIAL_ID}", variable: 'SONAR_TOKEN')
                ]) {
                    sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=wwp \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.token=${SONAR_TOKEN} \
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
                    echo "🔖 Project Version: ${env.ART_VERSION}"
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

                    echo "📦 Uploading WAR to Nexus..."

                    nexusArtifactUploader(
                        nexusVersion: "nexus3",
                        protocol: "http",
                        nexusUrl: "${NEXUS_URL}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        groupId: "koddas.web.war",
                        version: "${env.ART_VERSION}",
                        artifacts: [[
                            artifactId: "wwp",
                            classifier: "",
                            file: warFile,
                            type: "war"
                        ]]
                    )
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully!"

            echo "🔍 Sonar Dashboard:"
            echo "http://13.126.135.101:9000/dashboard?id=wwp"

            echo "📦 Nexus Artifact:"
            echo "http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/koddas/web/war/wwp/${env.ART_VERSION}/wwp-${env.ART_VERSION}.war"
        }

        failure {
            echo "❌ Pipeline failed. Check Sonar/Nexus logs."
        }
    }
}
