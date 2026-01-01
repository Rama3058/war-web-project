pipeline {
    agent any

    tools {
        maven "maven"
    }

    environment {
        // Nexus
        NEXUS_URL          = "15.207.55.128:8081"
        NEXUS_REPOSITORY   = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"

        // SonarQube server URL
        SONAR_HOST_URL = "http://13.126.135.101:9000"
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/mchetan-aradhya/war-web-project.git', branch: 'master'
            }
        }

        stage('Build WAR') {
            steps {
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.war'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                // Jenkins credential ID holding the real token as Secret Text
                SONAR_TOKEN = credentials('sonar_token')
            }
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
                    echo "Artifact Version: ${env.ART_VERSION}"
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

                    if (!warFile) {
                        error "WAR file not found in target directory"
                    }

                    echo "Uploading ${warFile} to Nexus"

                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUS_URL}",
                        groupId: 'koddas.web.war',
                        version: "${env.ART_VERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [[
                            artifactId: 'wwp',
                            classifier: '',
                            file: warFile,
                            type: 'war'
                        ]]
                    )
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
            echo "Nexus Artifact URL:"
            echo "http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/koddas/web/war/wwp/${env.ART_VERSION}/wwp-${env.ART_VERSION}.war"
        }
        failure {
            echo 'Pipeline failed. Check Jenkins logs.'
        }
    }
}
