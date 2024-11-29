pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: env.BRANCH_NAME, url: 'https://github.com/Rimonshil/DevSecOps-Project.git'
            }
        }
        stage('SonarQube Analysis') {
            when {
                anyOf {
                    branch 'main'
                    branch 'staging'
                }
            }
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=Netflix-${BRANCH_NAME} \
                    -Dsonar.projectKey=Netflix-${BRANCH_NAME}'''
                }
            }
        }
        stage('Quality Gate') {
            when {
                anyOf {
                    branch 'main'
                    branch 'staging'
                }
            }
            steps {
                script {
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "netflix:${BRANCH_NAME}"
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build --build-arg TMDB_V3_API_KEY=f204af711706e448c137300e98a0d6e3 -t ${imageTag} ."
                        sh "docker tag ${imageTag} 6164118899/devsecops:${BRANCH_NAME}"
                        sh "docker push 6164118899/devsecops:${BRANCH_NAME}"
                    }
                }
            }
        }
        stage('Security Scan with Trivy') {
            steps {
                sh "trivy image 6164118899/devsecops:${BRANCH_NAME} > trivy-${BRANCH_NAME}.txt"
            }
        }
        stage('Deploy Application') {
            when {
                branch 'main'
            }
            steps { 
                script {
                    sshagent(['prod-server']) {
                    sh '''
                    set -e
                    REMOTE_USER="azureuser"
                    REMOTE_SERVER="172.178.131.46"
                    SCRIPT_PATH="/home/azureuser/deploy-for-production.sh"
                    
                    echo "Deploying from script: ${SCRIPT_PATH} to ${REMOTE_SERVER}..."

                    # Run the deploy script on the production server
                    ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_SERVER} "bash ${SCRIPT_PATH}"
                    '''

                }                
            }
        }
    }
        stage('Deploy to Staging') {
            when {
                branch 'staging'
            }
            steps {
                sh 'docker run -d --name netflix -p 8081:80 6164118899/devsecops:staging'
            }
        }
    }
    post {
        success {
            echo "Pipeline for branch ${env.BRANCH_NAME} completed successfully."
        }
        failure {
            echo "Pipeline for branch ${env.BRANCH_NAME} failed."
        }
    }
}
