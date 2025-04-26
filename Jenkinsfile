pipeline {
    agent any

    tools {
        maven 'Maven 3'
    }

    environment {
        DOCKERHUB_USER = 'thuannguyen2905'
        DOCKER_CREDENTIALS_ID = 'docker-hub-credentials'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                checkout scm
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    def services = [
                        "spring-petclinic-customers-service",
                        "spring-petclinic-vets-service",
                        "spring-petclinic-visits-service",
                        "spring-petclinic-genai-service"
                    ]

                    def baseCommit = sh(script: "git rev-parse HEAD^1", returnStdout: true).trim()
                    def changedFiles = sh(script: "git diff --name-only ${baseCommit} HEAD", returnStdout: true).trim().split("\n")
                    echo "Changed files: ${changedFiles.join(', ')}"

                    def changedServices = []
                    for (service in services) {
                        if (changedFiles.any { it.startsWith(service) }) {
                            changedServices.add(service)
                        }
                    }
                    echo "Code changes in services: ${changedServices.join(', ')}"
                    env.CHANGED_SERVICES = changedServices.join(', ')
                }
            }
        }

        stage('Test Services') {
            when {
                expression { env.CHANGED_SERVICES != '' }
            }
            steps {
                script {
                    def changedServices = env.CHANGED_SERVICES.split(',')
                    changedServices.each { service -> 
                        echo "Testing service: ${service}"
                        dir("${env.WORKSPACE}/${service}") {
                            sh "mvn clean test surefire-report:report jacoco:report"
                            sh "cat target/surefire-reports/*.txt || true"
                            junit '**/target/surefire-reports/*.xml'
                            recordCoverage(
                                tools: [[parser: 'JACOCO', pattern: '**/target/site/jacoco/jacoco.xml']],
                                qualityGates: [
                                    [threshold: 70.0, metric: 'LINE', baseline: 'PROJECT', unstable: false]
                                ]
                            )
                            if (currentBuild.result == 'UNSTABLE' || currentBuild.result == 'FAILURE') {
                                error "Test stage failed due to test failures!"
                            }
                        }
                    }
                }
            }
        }

        stage('Build Services') {
            when {
                expression { env.CHANGED_SERVICES != '' }
            }
            steps {
                script {
                    def changedServices = env.CHANGED_SERVICES.split(',')
                    changedServices.each{ service -> 
                        echo "Building service: ${service}"
                        dir("${service}") {
                            sh "mvn clean package -DskipTests"
                        }
                    }
                }
            }
        }

        stage('Build and Push Docker Images') {
            when {
                expression { env.CHANGED_SERVICES != '' }
            }
            steps {
                script {
                    def commitId = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    def changedServices = env.CHANGED_SERVICES.split(',')

                    withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin'

                        changedServices.each { service ->
                            def shortName = service.replace('spring-petclinic-', '')
                            def imageName = "${DOCKERHUB_USER}/${shortName}:${commitId}"

                            echo "Building Docker image: ${imageName}"

                            dir(service) {
                                sh """
                                    docker build -t ${imageName} .
                                    docker push ${imageName}
                                """
                            }
                        }

                        // Optional: Logout after push
                        sh "docker logout"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline execution completed!"
        }
    }
}
