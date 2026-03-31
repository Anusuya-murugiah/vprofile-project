pipeline {

    agent any

    tools {
        maven "maven3"
    }

    environment {
        NEXUS_VERSION       = "nexus3"
        NEXUS_PROTOCOL      = "http"
        NEXUS_URL           = "172.31.40.209:8081"
        NEXUS_REPOSITORY    = "vprofile-release"
        NEXUS_CREDENTIAL_ID = "nexuslogin"

        ARTVERSION = "${env.BUILD_ID}"
        IMAGE_NAME = "vprofile-app"
        DOCKER_REGISTRY = "your-docker-repo"   // change this
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {

        stage('CHECKOUT') {
            steps {
                checkout scm
            }
        }

        stage('BUILD & TEST') {
            steps {
                sh 'mvn clean verify'
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('QUALITY CHECKS') {
            parallel {

                stage('CHECKSTYLE') {
                    steps {
                        sh 'mvn checkstyle:checkstyle'
                    }
                }

                stage('SONARQUBE ANALYSIS') {
                    environment {
                        scannerHome = tool 'sonarscanner4'
                    }
                    steps {
                        withSonarQubeEnv('sonar-pro') {
                            sh '''
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=vprofile \
                            -Dsonar.projectName=vprofile-repo \
                            -Dsonar.sources=src \
                            -Dsonar.java.binaries=target/classes \
                            -Dsonar.junit.reportsPath=target/surefire-reports \
                            -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                            -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                            '''
                        }
                    }
                }
            }
        }

        stage('QUALITY GATE') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('PUBLISH TO NEXUS') {
            steps {
                script {
                    def pom = readMavenPom file: "pom.xml"
                    def files = findFiles(glob: "target/*.${pom.packaging}")

                    if (files.length > 0) {
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: ARTVERSION,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [
                                    artifactId: pom.artifactId,
                                    classifier: '',
                                    file: files[0].path,
                                    type: pom.packaging
                                ],
                                [
                                    artifactId: pom.artifactId,
                                    classifier: '',
                                    file: "pom.xml",
                                    type: "pom"
                                ]
                            ]
                        )
                    } else {
                        error "Artifact not found!"
                    }
                }
            }
        }

        stage('BUILD DOCKER IMAGE') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${ARTVERSION} ."
                }
            }
        }

        stage('PUSH DOCKER IMAGE') {
            steps {
                script {
                    sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${ARTVERSION}"
                }
            }
        }

        stage('DEPLOY') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh '''
                    kubectl set image deployment/vprofile \
                    vprofile=${DOCKER_REGISTRY}/${IMAGE_NAME}:${ARTVERSION}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
        always {
            cleanWs()
        }
    }
}
