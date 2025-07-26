pipeline {
    agent any

    environment { 
        GIT_REPO    = 'https://github.com/YongSinh/agritech-iot.git'
        APP_ENV     = 'uat'
        APP_VERSION = '1.0.0'
    }

    parameters {
        choice(name: 'BRANCH', choices: ['dev', 'uat', 'main'], description: 'Please select branch')
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
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
                    def projectDir = "iot-app-react"
                    def imageName = "react-app-iot:1.0.0"

                    echo "🔧 Building Docker image: ${imageName}"

                    // Build the Docker image
                    sh """
                        cd ${projectDir}
                        DOCKER_BUILDKIT=1 docker build --progress=plain -t ${imageName} . | tee docker-build.log
                    """

                    env.DOCKER_IMAGE = imageName
                }
            }
        }

        stage('Deploy Project') {
            steps {
                script {
                    echo "🚀 Running Docker container from image: react-app-iot:1.0.0"

                    // Stop and remove existing container if it exists (optional cleanup)
                    sh """
                        docker rm -f react-app-iot || true
                        docker run -d --name react-app-iot -p 5173:443 react-app-iot:1.0.0
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build and deployment completed successfully."
        }
        failure {
            echo "❌ Build or deployment failed. Check the logs for details."
        }
    }
}
