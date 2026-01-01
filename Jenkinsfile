pipeline {
    agent any
    
    environment {
        // --- Nexus Config ---
        NEXUS_URL = "15.207.55.128:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
        
        // --- SonarQube Config ---
        SONAR_HOST_URL = "http://13.126.135.101:9000"
        SONAR_CRED_ID = "SONAR_TOKEN" 
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
                echo "🔍 Running Static Code Analysis..."
                // Using your specific credential ID: SONAR_TOKEN
                withCredentials([string(credentialsId: "${env.SONAR_CRED_ID}", variable: 'S_TOKEN')]) {
                    sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=wwp \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.token=${S_TOKEN} \
                        -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }
        
        stage('Extract Version') {
            steps {
                script {
                    // Pulls version from pom.xml dynamically
                    env.ART_VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()
                    echo "🔖 Detected Project Version: ${env.ART_VERSION}"
                }
            }
        }
        
        stage('Publish to Nexus') {
            steps {
                script {
                    // Dynamically find the path of the generated war file
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
                    
                    echo "🚀 Uploading to Nexus: ${warFile}"
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUS_URL}",
                        groupId: 'koddas.web.war',
                        version: "${env.ART_VERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
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
    
    post {
        success {
            echo "✅ Success! Artifact uploaded to Nexus."
            echo "URL: http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/koddas/web/war/wwp/${env.ART_VERSION}/"
        }
        failure {
            echo "❌ Pipeline failed. Check the stage logs above for details."
        }
    }
}
