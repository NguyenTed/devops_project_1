pipeline {
    agent any
    
    tools {
        maven 'Maven 3' // Use Jenkins' built-in Maven
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

                    // Fetch everything, ensuring full history is available
                    sh 'git fetch --all --prune'

                    // Get the current branch name
                    def currentBranch = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                    echo "Current branch: ${currentBranch}"

                    // Determine the base branch (fallback to main)
                    def baseBranch = env.CHANGE_TARGET ?: 'main'
                    echo "Comparing changes against: ${baseBranch}"

                    // Ensure the base branch is available locally
                    sh "git checkout ${baseBranch} || git checkout -b ${baseBranch} origin/${baseBranch} || true"

                    // Get latest common commit between the current branch and the base branch
                    def baseCommit = sh(script: "git merge-base origin/${baseBranch} HEAD || echo $(git rev-list --max-parents=0 HEAD)", returnStdout: true).trim()
                    echo "Base commit: ${baseCommit}"

                    // Get the list of changed files
                    def changedFiles = sh(script: "git diff --name-only ${baseCommit} HEAD || true", returnStdout: true).trim().split("\n")

                    // Identify changed services
                    def changedServices = []
                    for (service in services) {
                        if (changedFiles.any { it.startsWith(service) }) {
                            changedServices.add(service)
                        }
                    }

                    echo "Code changes detected in branch '${currentBranch}': ${changedServices.join(', ')}"
                    env.CHANGED_SERVICES = changedServices.join(', ')
                    env.CURRENT_BRANCH = currentBranch
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
                    changedServices.each{ service -> 
                        echo "Testing service: ${service}"
                        dir("${service}") {
                            // Run tests and generate coverage reports
                            sh "mvn test surefire-report:report jacoco:report"
        
                            // Publish JUnit test results
                            junit '**/target/surefire-reports/*.xml'
        
                            // Record test coverage using the Coverage plugin
                            recordCoverage(
                                tools: [[parser: 'JACOCO', pattern: '**/target/site/jacoco/jacoco.xml']],
                                qualityGates: [
                                    [threshold: 70.0, metric: 'LINE', baseline: 'PROJECT', unstable: false]
                                ]
                            )

                            // Check if build is unstable and force failure
                            if (currentBuild.result == 'UNSTABLE') {
                                error "Test coverage is below 70%, failing the build!"
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
    }
    
    post {
        always {
            echo "Pipeline execution completed!"
        }
    }
}
