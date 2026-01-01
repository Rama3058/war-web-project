pipeline {
    agent any

    environment {
        // -------- SonarQube --------
        SONAR_HOST_URL = "http://13.126.135.101:9000"

        // -------- Nexus --------
        NEXUS_URL = "15.207.55.128:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
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
                        script: 'find target -name "*.war" -print -quit',
                        returnStdout: true
                    ).trim()

                    echo "🚀 Uploading WAR to Nexus..."

                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "15.207.55.128:8081",
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
            echo "🔗 SonarQube Dashboard: http://13.126.135.101:9000/dashboard?id=wwp"
            echo "📦 Nexus: http://15.207.55.128:8081"
        }
        failure {
            echo "❌ Pipeline failed — check logs"
        }
    }
}
