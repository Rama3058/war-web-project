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
        TOMCAT_URL = "http://43.204.235.239:8080"
        TOMCAT_CREDENTIAL_ID = "tomcat_creds"
        TOMCAT_CONTEXT = "wwp"
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

                    def releaseVersion = "${ART_VERSION}-${BUILD_NUMBER}"

                    echo "🚀 Uploading WAR to Nexus"

                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUS_URL}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        groupId: 'koddas.web.war',
                        version: releaseVersion,
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

        stage('Deploy to Tomcat') {
            steps {
                echo "🚀 Deploying WAR to Tomcat..."

                withCredentials([
                    usernamePassword(
                        credentialsId: "${TOMCAT_CREDENTIAL_ID}",
                        usernameVariable: 'TOMCAT_USER',
                        passwordVariable: 'TOMCAT_PASS'
                    )
                ]) {
                    sh '''
                        WAR_FILE=$(find target -name "*.war" -print -quit)

                        echo "➡ Undeploying existing app (if any)"
                        curl -u $TOMCAT_USER:$TOMCAT_PASS \
                        "$TOMCAT_URL/manager/text/undeploy?path=/$TOMCAT_CONTEXT" || true

                        echo "➡ Deploying new WAR"
                        curl -u $TOMCAT_USER:$TOMCAT_PASS \
                        -T $WAR_FILE \
                        "$TOMCAT_URL/manager/text/deploy?path=/$TOMCAT_CONTEXT&update=true"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully"
            echo "🔗 SonarQube Dashboard: ${SONAR_HOST_URL}/dashboard?id=wwp"
            echo "📦 Nexus Repository: http://${NEXUS_URL}"
            echo "🌐 Application URL: ${TOMCAT_URL}/${TOMCAT_CONTEXT}"
        }

        failure {
            echo "❌ Pipeline failed — check logs for details"
        }

        always {
            echo "🧹 Pipeline execution finished"
        }
    }
}
