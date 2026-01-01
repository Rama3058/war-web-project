pipeline {
    agent any

    environment {
        // -------- SonarQube --------
        SONAR_HOST_URL = "http://13.126.135.101:9000"
        SONAR_CREDENTIAL_ID = "SONAR_TOKEN"

        // -------- Nexus --------
        NEXUS_URL = "15.207.55.128:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"

        // -------- Tomcat --------
        TOMCAT_SERVER = "43.204.112.166"
        TOMCAT_USER = "ubuntu"
        SSH_KEY_PATH = "/var/lib/jenkins/.ssh/jenkins_key"
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
                    string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')
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

        stage('Deploy to Tomcat') {
            steps {
                script {
                    def warFile = sh(
                        script: 'find target -name "*.war" -print -quit',
                        returnStdout: true
                    ).trim()

                    echo "🚀 Deploying to Tomcat..."

                    sh """
                        scp -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ${warFile} ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/
                        ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_SERVER} '
                            sudo mv /tmp/*.war /opt/tomcat/webapps/ &&
                            sudo systemctl restart tomcat
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully"
            echo "🔗 Sonar: http://13.126.135.101:9000"
            echo "📦 Nexus: http://15.207.55.128:8081"
            echo "🌐 App: http://${TOMCAT_SERVER}:8080/wwp"
        }
        failure {
            echo "❌ Pipeline failed — check logs"
        }
    }
}
