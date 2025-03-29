pipeline {
    agent any

    tools {
        maven "maven"
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "3.109.203.221:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
        ART_VERSION = "1.0.0"
        TOMCAT_URL = "http://43.204.112.166:8080"
        TOMCAT_CREDENTIAL_ID = "tomcat_creds"  // Using the added credential ID
    }

    stages {
        stage('Build WAR') {
            steps {
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: '**/target/*.war'
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
                    def pomFile = "pom.xml"

                    if (!fileExists(warFile)) {
                        error("WAR file not found at ${warFile}")
                    }
                    if (!fileExists(pomFile)) {
                        error("POM file not found.")
                    }

                    nexusArtifactUploader(
                        nexusVersion: "${NEXUS_VERSION}",
                        protocol: "${NEXUS_PROTOCOL}",
                        nexusUrl: "${NEXUS_URL}",
                        groupId: "com.example.warwebproject",
                        version: "${ART_VERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [
                            [artifactId: "wwp", classifier: '', file: warFile, type: "war"],
                            [artifactId: "wwp", classifier: '', file: pomFile, type: "pom"]
                        ]
                    )
                }
            }
        }

        stage('Deploy WAR') {
            steps {
                script {
                    def warFilePath = sh(script: "find ${workspace}/target -name '*.war' -type f -print0 | xargs -0 ls -t | head -n 1", returnStdout: true).trim()

                    if (warFilePath) {
                        echo "WAR file located at: ${warFilePath}"

                        withCredentials([usernamePassword(credentialsId: "${TOMCAT_CREDENTIAL_ID}", usernameVariable: 'TOMCAT_USER', passwordVariable: 'TOMCAT_PASS')]) {
                            sh """
                                scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/jenkins_key ${warFilePath} ${TOMCAT_USER}@43.204.112.166:/tmp/wwp.war

                                ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/jenkins_key ${TOMCAT_USER}@43.204.112.166 <<EOF
                                    set -e
                                    echo "Moving WAR file to Tomcat directory..."
                                    sudo mv /tmp/wwp.war /opt/tomcat/webapps/wwp.war
                                    echo "WAR file deployed successfully."

                                    echo "Restarting Tomcat..."
                                    echo ${TOMCAT_PASS} | sudo -S systemctl restart tomcat
                                    echo "Tomcat restarted."
                                EOF
                            """
                        }
                    } else {
                        error "No WAR file found to deploy."
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
        }
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed. Check logs for details.'
        }
    }
}
