    pipeline {
        agent any
        environment { 
            GIT_REPO = 'https://gitlab.phillipbank.com.kh/integration/innovation-apache-camel.git'
            Telegram_Token = '7837537423:AAH_HMREL7DFFmi3oyQNXzdANHe4tLeUgv4'  // Use Jenkins credentials for your Telegram token
            Telegram_ChatID = '-4772195077'    // Use Jenkins credentials for your chat ID
            APP_ENV= 'uat'
            APP_VERSION= '1.0.0'
        }
        parameters {
            choice(name: 'BRANCH', choices: ['main', 'uat'], description: 'Please select branch')
            choice(name: 'PROJECT', choices: [
                'errorhandling', 
                'eda', 
                'load-balancing', 
                'openapi-gateway', 
                'protocol', 
                'schedule', 
                'orchestration',
                'mediation'
            ], description: 'Please select project')
        }
        tools {
            jdk 'open-jdk-17'
            maven 'Maven 3.9.5' // Make sure Maven is set up in Jenkins
        }
        stages {
            stage('Clean Workspace') {
                steps {
                    cleanWs() // Clean the workspace before starting the build
                }
            }
            stage('Checkout Code') {
                steps {
                    script {
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: "*/${params.BRANCH}"]],
                            extensions: [
                                [$class: 'SparseCheckoutPaths', sparseCheckoutPaths: [[path: "${params.PROJECT}"]]]
                            ],
                            userRemoteConfigs: [[
                                url: "${env.GIT_REPO}",
                                credentialsId: 'jenkins_ci'
                            ]]
                        ])
                    }
                }
            }
            stage('Build Project') {
                steps {
                    script {
                        def projectDir = "${params.PROJECT}"
                        def pomFile = "${projectDir}/pom.xml"
                        
                        if (!fileExists(pomFile)) {
                            error "POM file not found at '${pomFile}'. Please check the repository structure."
                        }
                        
                        // Capture Maven output
                        def mavenOutput = sh(script: """
                            cd ${projectDir}
                            echo "Building project: ${projectDir}"
                            mvn -f pom.xml clean package spring-boot:build-image -DskipTests=true
                        """, returnStdout: true).trim()

                        // Extract Docker image name from Maven output
                        def imageName = (mavenOutput =~ /Successfully built image '([^']+)'/)[0][1]
                        echo "Built Docker image: ${imageName}"

                        // Store the image name in an environment variable for later use
                        env.DOCKER_IMAGE = imageName
                    }
                }
            }
        }
        stages {
            stage('SonarQube Analysis') {
                steps {
                    script {
                        // Mapping Project to SonarQube Project Key
                        def sonarProjectKeys = [
                            'errorhandling'     : 'integration_errorhandling_KEY',
                            'eda'               : 'integration_eda_KEY',
                            'load-balancing'    : 'integration_loadbalancing_KEY',
                            'openapi-gateway'   : 'integration_openapi_KEY',
                            'protocol'          : 'integration_protocol_KEY',
                            'schedule'          : 'integration_schedule_KEY',
                            'orchestration'     : 'integration_orchestration_KEY',
                            'mediation'         : 'integration_mediation_KEY'
                        ]

                        def selectedSonarKey = sonarProjectKeys[params.PROJECT]
                        
                        if (!selectedSonarKey) {
                            error "SonarQube project key not found for selected project: ${params.PROJECT}"
                        }

                        echo "Running SonarQube analysis for project: ${params.PROJECT} with key: ${selectedSonarKey}"

                        sh """
                            cd ${params.PROJECT}
                            mvn clean verify sonar:sonar \\
                                -Dsonar.projectKey=${selectedSonarKey} \\
                                -Dsonar.host.url=${env.SONAR_HOST_URL} \\
                                -Dsonar.login=${env.SONAR_LOGIN} \\
                                -Ddockerfile.skip=true \\
                                -Dsonar.coverage.jacoco.xmlReportPaths=target/jacoco-report/jacoco.xml \\
                                -P uat
                        """
                    }
                }
            }
        }
       post {
            success {
                script {
                    echo "✅ Build completed successfully for project: ${params.PROJECT}"
                    def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId
                    def buildNumber = currentBuild.number
                    
                    // Get current date and time when the build started
                    def buildStartTimeMillis = currentBuild.getStartTimeInMillis()
                    def buildStartDate = new Date(buildStartTimeMillis)
                    def formattedBuildTime = buildStartDate.format("yyyy-MM-dd HH:mm:ss")
                    def buildFinishTime = new Date().format("yyyy-MM-dd HH:mm:ss")
                    echo "Build finished at: ${buildFinishTime}"
                    def consoleUrl = "${env.BUILD_URL}console"
                    sh """
                        curl -s -X POST https://api.telegram.org/bot$Telegram_Token/sendMessage \
                            -d chat_id=${Telegram_ChatID} \
                            -d parse_mode="HTML" \
                            -d disable_web_page_preview=true \
                            -d text="<b>Stage</b>: Build project ${params.PROJECT} \
                            %0A<b>Build Number</b>: ${buildNumber} \
                            %0A<b>Build Start</b>: ${formattedBuildTime} \
                            %0A<b>Build Finish</b>: ${buildFinishTime} \
                            %0A<b>Status</b>: ✅ ${currentBuild.currentResult} \
                            %0A<b>Version</b>: ${env.APP_ENV}-${env.APP_VERSION} \
                            %0A<b>Environment</b>: ${env.APP_ENV} \
                            %0A<b>Image</b>: ${env.DOCKER_IMAGE ?: 'Unknown Image'} \
                            %0A<b>Console Log</b>: <a href='${consoleUrl}'>View Logs</a> \
                            %0A<b>User Build</b>: ${buildUser ?: 'Unknown'}"
                    """
                }
            }
            failure {
                script {
                    echo "❌ Build failed for project: ${params.PROJECT}. Check the logs for details."
                    def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId
                    def buildNumber = currentBuild.number
                    
                    // Get current date and time when the build started
                    def buildStartTimeMillis = currentBuild.getStartTimeInMillis()
                    def buildStartDate = new Date(buildStartTimeMillis)
                    def formattedBuildTime = buildStartDate.format("yyyy-MM-dd HH:mm:ss")
                    def buildFinishTime = new Date().format("yyyy-MM-dd HH:mm:ss")
                    echo "Build finished at: ${buildFinishTime}"
                    def consoleUrl = "${env.BUILD_URL}console"
                    sh """
                        curl -s -X POST https://api.telegram.org/bot$Telegram_Token/sendMessage \
                            -d chat_id=${Telegram_ChatID} \
                            -d parse_mode="HTML" \
                            -d disable_web_page_preview=true \
                            -d text="<b>Stage</b>: Build project ${params.PROJECT} \
                            %0A<b>Build Number</b>: ${buildNumber} \
                            %0A<b>Build Start</b>: ${formattedBuildTime} \
                            %0A<b>Build Finish</b>: ${buildFinishTime} \
                            %0A<b>Status</b>: ❌ ${currentBuild.currentResult} \
                            %0A<b>Version</b>: ${env.APP_ENV}-${env.APP_VERSION} \
                            %0A<b>Environment</b>: ${env.APP_ENV} \
                            %0A<b>Image</b>: ${env.DOCKER_IMAGE ?: 'Unknown Image'} \
                            %0A<b>Console Log</b>: <a href='${consoleUrl}'>View Logs</a> \
                            %0A<b>User Build</b>: ${buildUser ?: 'Unknown'}"
                    """
                }
            }
        }
    }
