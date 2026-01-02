pipeline {
    agent any

    environment {
        // -------- SonarQube --------
        SONAR_HOST_URL = "http://13.126.135.101:9000"

        // -------- Nexus --------
        NEXUS_URL = "15.207.55.128:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
        // ---- Server & Credentials ----
        TOMCAT_SERVER = "43.204.235.239"
        TOMCAT_USER = "ubuntu"
        SSH_KEY_PATH = "/var/lib/jenkins/.ssh/jenkins_key"
    }

    tools {
        maven "maven"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build WAR') {
            steps {
                echo "🔨 Building WAR..."
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.war'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "🔍 Running SonarQube analysis..."

                withCredentials([
                    usernamePassword(
                        credentialsId: 'sonar_creds',
                        usernameVariable: 'SONAR_USER',
                        passwordVariable: 'SONAR_TOKEN'
                    )
                ]) {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=wwp \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.token=${SONAR_TOKEN} \
                        -Dsonar.java.binaries=target/classes
                    '''
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
                    echo "📦 Artifact Version: ${ART_VERSION}"
                }
            }
        }

        
 stage('Publish to Nexus') {
            steps {
                script {
                    def warFile = sh(
                        script: "find target -name '*.war' -print -quit",
                        returnStdout: true
                    ).trim()

                    // Ensure unique version for release repository
                    def releaseVersion = "${ART_VERSION}-${BUILD_NUMBER}"

                    echo "🚀 Uploading WAR to Nexus"
                    echo "📦 Version: ${releaseVersion}"
                    echo "📄 File: ${warFile}"

                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUS_URL}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        groupId: 'koddas.web.war',
                        version: releaseVersion,
                        artifacts: [
                            [
                                artifactId: 'wwp',
                                classifier: '',
                                file: warFile,
                                type: 'war'
                            ]
                        ]
                    )
                }
            }
        }
    }
stage('Deploy to Tomcat') {
            steps {
                script {
                    echo "🚀 Deploying WAR to Tomcat server..."
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()

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
    
    post {
        success {
            echo "✅ Pipeline completed successfully"
            echo "🔗 SonarQube Dashboard: ${SONAR_HOST_URL}/dashboard?id=wwp"
            echo "📦 Nexus Repository: http://${NEXUS_URL}"
        }

        failure {
            echo "❌ Pipeline failed — check logs for details"
        }

        always {
            echo "🧹 Pipeline execution finished"
        }
    }
}
