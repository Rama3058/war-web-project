pipeline {
    agent any

    environment {
        // -------- SonarQube --------
        SONAR_HOST_URL = "http://13.126.135.101:9000"

        // -------- Nexus --------
        NEXUS_URL = "15.207.55.128:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"

        // -------- Tomcat --------
        TOMCAT_SERVER = "43.204.235.239"
        TOMCAT_USER   = "ubuntu"
        TOMCAT_WEBAPPS = "/opt/tomcat/webapps"
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

        // stage('SonarQube Analysis') {
        //     steps {
        //         echo "🔍 Running SonarQube analysis..."

        //         withCredentials([
        //             usernamePassword(
        //                 credentialsId: 'sonar_creds',
        //                 usernameVariable: 'SONAR_USER',
        //                 passwordVariable: 'SONAR_TOKEN'
        //             )
        //         ]) {
        //             sh '''
        //                 mvn sonar:sonar \
        //                   -Dsonar.projectKey=wwp \
        //                   -Dsonar.host.url=${SONAR_HOST_URL} \
        //                   -Dsonar.token=${SONAR_TOKEN} \
        //                   -Dsonar.java.binaries=target/classes
        //             '''
        //         }
        //     }
        // }

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

        // stage('Publish to Nexus') {
        //     steps {
        //         script {
        //             def warFile = sh(
        //                 script: "find target -name '*.war' -print -quit",
        //                 returnStdout: true
        //             ).trim()

        //             echo "🚀 Uploading WAR to Nexus..."

        //             nexusArtifactUploader(
        //                 nexusVersion: 'nexus3',
        //                 protocol: 'http',
        //                 nexusUrl: "${NEXUS_URL}",
        //                 repository: "${NEXUS_REPOSITORY}",
        //                 credentialsId: "${NEXUS_CREDENTIAL_ID}",
        //                 groupId: 'koddas.web.war',
        //                 artifactId: 'wwp',
        //                 version: "${ART_VERSION}",
        //                 artifacts: [
        //                     [artifactId: 'wwp', file: warFile, type: 'war']
        //                 ]
        //             )
        //         }
        //     }
        // }

        stage('Deploy to Tomcat') {
            steps {
                echo "🚀 Deploying to Tomcat..."

                sshagent(credentials: ['tomcat_ssh_key']) {
                    sh '''
                        WAR_FILE=$(ls target/*.war)

                        scp -o StrictHostKeyChecking=no $WAR_FILE ${TOMCAT_USER}@${TOMCAT_SERVER}:/tmp/

                        ssh -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_SERVER} << EOF
                          sudo systemctl stop tomcat || true
                          sudo rm -rf ${TOMCAT_WEBAPPS}/*.war ${TOMCAT_WEBAPPS}/*
                          sudo mv /tmp/*.war ${TOMCAT_WEBAPPS}/
                          sudo systemctl start tomcat
                        EOF
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully"
            echo "🔗 SonarQube: ${SONAR_HOST_URL}/dashboard?id=wwp"
            echo "📦 Nexus: http://${NEXUS_URL}"
            echo "🌐 App URL: http://${TOMCAT_SERVER}:8080/wwp"
        }
        failure {
            echo "❌ Pipeline failed — check logs"
        }
    }
}
