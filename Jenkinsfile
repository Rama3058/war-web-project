pipeline {
    agent any

    environment {
        TOMCAT_SERVER = "43.204.112.166"
        TOMCAT_USER   = "ubuntu"

        NEXUS_URL = "172.31.3.179:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"

        SSH_KEY_PATH = "/var/lib/jenkins/.ssh/jenkins_key"

        // ⚠ TEMPORARY – Hard-coded SonarQube details
        SONAR_HOST_URL = "http://15.206.195.36:9000"
        SONAR_TOKEN    = "squ_8a380c0d68321708030eff61650223efd226f0d9"
    }

    tools {
        maven "maven"
    }

    stages {

        stage('Build WAR') {
            steps {
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: '**/target/*.war'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                sh """
                    mvn sonar:sonar \
                      -Dsonar.projectKey=wwp \
                      -Dsonar.host.url=${SONAR_HOST_URL} \
                      -Dsonar.token=${SONAR_TOKEN} \
                      -Dsonar.java.binaries=target/classes
                """
            }
        }

        stage('Extract Version') {
            steps {
                script {
                    env.ART_VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()
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

                    nexusArtifactUploader(
                        nexusVersion: "nexus3",
                        protocol: "http",
                        nexusUrl: "${NEXUS_URL}",
                        groupId: "koddas.web.war",
                        artifactId: "wwp",
                        version: "${ART_VERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [[
                            file: warFile,
                            type: "war"
                        ]]
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

                    sh """
                        scp -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no \
                            ${warFile} ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/

                        ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no \
                            ${TOMCAT_USER}@${TOMCAT_SERVER} '
                            sudo mv /tmp/*.war /opt/tomcat/webapps/ &&
                            sudo systemctl restart tomcat'
                    """
                }
            }
        }

        stage('Display URLs') {
            steps {
                script {
                    echo "🌐 App URL   : http://${TOMCAT_SERVER}:8080/wwp-${ART_VERSION}"
                    echo "📦 Nexus URL: http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/koddas/web/war/wwp/${ART_VERSION}/wwp-${ART_VERSION}.war"
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Check logs.'
        }
    }
}
