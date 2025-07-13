pipeline {
    agent any

    environment { 
        GIT_REPO   = 'https://github.com/YongSinh/agritech-iot.git'
        APP_ENV    = 'uat'
        APP_VERSION = '1.0.0'
    }

    parameters {
        choice(name: 'BRANCH', choices: ['main', 'uat', 'dev'], description: 'Please select branch')
        // Changed to extendedChoice for multi-select
        extendedChoice(
            name: 'PROJECTS', 
            type: 'CHECKBOX', 
            description: 'Select one or more projects to build',
            value: 'agritech-iot,logs-service,api-gateway',
            defaultValue: 'agritech-iot',
            multiSelectDelimiter: ','
        )
        choice(name: 'JDK', choices: ['jdk-17.0.12', 'graalvm-jdk-21', 'graalvm-jdk-24'], description: 'Please select JDK version')
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
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.BRANCH}"]],
                        extensions: [
                            [$class: 'SparseCheckoutPaths', sparseCheckoutPaths: [[path: "."]]]
                        ],
                        userRemoteConfigs: [[
                            url: "${env.GIT_REPO}",
                            credentialsId: 'jenkins_ci'
                        ]]
                    ])
                }
            }
        }

        stage('Build Projects') {
            steps {
                script {
                    // Convert the comma-separated string to a list
                    def selectedProjects = params.PROJECTS.split(',')
                    def builtImages = [:]
                    
                    // Build projects in parallel
                    def parallelStages = [:]
                    
                    selectedProjects.each { project ->
                        parallelStages["Build ${project}"] = {
                            try {
                                def projectDir = "${project}"
                                def pomFile = "${projectDir}/pom.xml"
                                
                                if (!fileExists(pomFile)) {
                                    error "POM file not found at '${pomFile}'"
                                }
                                
                                def mavenOutput = sh(script: """
                                    cd ${projectDir}
                                    echo "Building project: ${project}"
                                    mvn -f pom.xml clean package spring-boot:build-image -DskipTests=true
                                """, returnStdout: true).trim()

                                def matcher = (mavenOutput =~ /Successfully built image '([^']+)'/)
                                if (matcher.find()) {
                                    def imageName = matcher[0][1]
                                    builtImages[project] = imageName
                                    echo "✅ Successfully built Docker image for ${project}: ${imageName}"
                                } else {
                                    error "❌ Could not extract image name from Maven output for ${project}"
                                }
                            } catch (Exception e) {
                                error "❌ Failed to build ${project}: ${e.message}"
                            }
                        }
                    }
                    
                    parallel parallelStages
                    
                    // Store all built images for deployment
                    env.BUILT_IMAGES = builtImages.inspect()
                }
            }
        }

        stage('Deploy Projects') {
            steps {
                script {
                    def builtImages = evaluate(env.BUILT_IMAGES)
                    
                    if (!builtImages) {
                        error "No images were built successfully"
                    }
                    
                    echo "✅ The following images are ready for deployment:"
                    builtImages.each { project, image ->
                        echo "  - ${project}: ${image}"
                        // Add your deployment logic here for each image
                        // Example: 
                        // sh "docker push ${image}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build completed successfully for projects: ${params.PROJECTS}"
        }
        failure {
            echo "❌ Build failed for one or more projects. Check the logs for details."
        }
    }
}