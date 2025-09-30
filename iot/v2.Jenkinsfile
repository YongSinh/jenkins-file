pipeline {
    agent any

    environment { 
        GIT_REPO    = 'https://github.com/YongSinh/agritech-iot.git'
        APP_ENV     = 'uat'
        APP_VERSION = '1.0.0'
    }

    parameters {
        booleanParam(name: 'BUILD_AGRITECH_IOT', defaultValue: false, description: 'Build agritech-iot project')
        booleanParam(name: 'BUILD_LOGS_SERVICE', defaultValue: false, description: 'Build logs-service project')
        booleanParam(name: 'BUILD_API_GATEWAY', defaultValue: false, description: 'Build api-gateway project')
        choice(name: 'BRANCH', choices: ['dev', 'uat', 'main'], description: 'Please select branch')
        choice(name: 'PROFILE', choices: ['dev', 'uat', 'prod'], description: 'Please select profile')
        choice(name: 'JDK', choices: ['graalvm-jdk-21', 'graalvm-jdk-24'], description: 'Please select JDK version')
    }

    tools {
        jdk "${params.JDK}"
        maven 'Maven 3.9.5'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Validate Tools') {
            steps {
                script {
                    def javaHome = tool name: "${params.JDK}", type: 'jdk'
                    def javaPath = "${javaHome}/bin/java"

                    if (!fileExists(javaPath)) {
                        error("❌ JDK not found or incompatible at: ${javaPath}")
                    }

                    def javaVersion = sh(script: "${javaPath} -version 2>&1", returnStdout: true).trim()
                    echo "✅ Using Java: ${javaVersion}"
                }
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: "${params.BRANCH}",
                    url: "${env.GIT_REPO}",
                    credentialsId: 'jenkins_ci'
            }
        }

        stage('Build Project') {
            steps {
                script {
                    def projects = [
                        'agritech-iot': params.BUILD_AGRITECH_IOT,
                        'logs-service': params.BUILD_LOGS_SERVICE,
                        'api-gateway': params.BUILD_API_GATEWAY
                    ]
                    def profile = "${params.PROFILE}"
                    def selectedProjects = projects.findAll { it.value }.keySet()

                    if (selectedProjects.isEmpty()) {
                        error "❌ No projects selected for build!"
                    }

                    echo "🔨 Building selected projects: ${selectedProjects.join(', ')}"

                    selectedProjects.each { project ->
                        echo "➡️ Starting build for: ${project}"

                        def projectDir = "${project}"
                        def pomFile = "${projectDir}/pom.xml"

                        if (!fileExists(pomFile)) {
                            error "❌ POM file not found at '${pomFile}'. Please check the repository structure."
                        }

                        dir(projectDir) {
                            echo "📦 Building project: ${projectDir}"

                            // Run Maven with live logs and capture output in a file
                            sh """
                                mvn -f pom.xml clean package spring-boot:build-image \
                                    -DskipTests=true -P ${profile} 
                            """
                        }
                    }
                }
            }
        }


        stage('Delpoy Project') {
            steps {
                script {
                    def composeFile = "docker-compose.yml"
            
                    // Check if docker-compose file exists
                    if (!fileExists(composeFile)) {
                        error "❌ docker-compose.yml not found in workspace."
                    }

                    // Print confirmation
                    echo "📦 Starting Docker Compose deployment..."
                    
                    // Stop and remove previous containers
                    sh "docker-compose down"

                    // Deploy using docker-compose
                    sh "docker-compose up -d --build"

                    echo "✅ Docker Compose deployment completed."
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build completed successfully."
        }
        failure {
            echo "❌ Build failed. Check the logs for details."
        }
    }
}
