pipeline {
    agent any
    
    environment {
        // Nexus Configuration
        NEXUS_URL = "15.207.55.128:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
        
        // SonarQube Configuration
        SONAR_HOST_URL = "http://13.126.135.101:9000"
        // This is the ID of the Secret Text credential you created in Jenkins
        SONAR_AUTH_TOKEN_ID = "sonar_token_cred" 
    }
    
    tools {
        maven "maven"
    }
    
    stages {
        stage('Build WAR') {
            steps {
                // Ensure we start clean
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.war', allowEmptyArchive: false
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                // withCredentials safely masks the token in logs
                withCredentials([string(credentialsId: "${SONAR_AUTH_TOKEN_ID}", variable: 'S_TOKEN')]) {
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
                    // Extract version from pom.xml
                    env.ART_VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()
                    echo "🔖 Detected Version: ${env.ART_VERSION}"
                }
            }
        }
        
        stage('Publish to Nexus') {
            steps {
                script {
                    // Locate the WAR file dynamically
                    def warFile = sh(script: 'ls target/*.war', returnStdout: true).trim()
                    
                    echo "📦 Uploading to Nexus: ${warFile}"
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
            echo '✅ Pipeline completed successfully!'
            echo "URL: http://${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/koddas/web/war/wwp/${env.ART_VERSION}/"
        }
        failure {
            echo '❌ Pipeline failed. Please check the stage logs above.'
        }
    }
}
