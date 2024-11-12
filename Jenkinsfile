pipeline {
    agent any
    tools {
        jdk 'jdk'
        nodejs 'nodejs-16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        AWS_ACCOUNT_ID = credentials('ACCOUNT_ID')
        AWS_ECR_REPO_NAME = credentials('ECR_REPO2')
        AWS_DEFAULT_REGION = 'ap-south-1'
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/"
    }
    parameters {
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip tests')
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout SCM') {
            steps {
                git url: 'https://github.com/Akshaya-Parthi/web-app-example.git', branch: 'main'
            }
        }
        stage('Install Dependencies') {
            steps {
                dir('api') {
                    sh 'npm install'
                }
            }
        }
        stage('Run Tests') {
            when {
                expression { return !params.SKIP_TESTS }  
            }
            steps {
                dir('api') {
                    sh """
                    python3 test.py
                    """
                }
            }
        }
        stage('Sonarqube Analysis') {
            steps {
                dir('api') {
                    withSonarQubeEnv('sonar-server') {
                        sh '''${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=api \
                        -Dsonar.projectKey=api'''
                    }
                }
            }
        }
        stage('Quality Check') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            }
        }
        stage("OWASP Dependency Check"){
            steps{
                dependencyCheck additionalArguments: '--scan ./ --format XML ', odcInstallation: 'DP'
                dependencyCheckPublisher pattern: '**/api-dependency-check-report.xml'
            }
        }
        stage("Docker Image Build") {
            steps {
                script {
                    dir('api') {
                            sh 'docker build -t ${AWS_ECR_REPO_NAME} .'
                    }
                }
            }
        }
        stage("ECR Image Pushing") {
            steps {
                script {
                        sh 'aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}'
                        sh 'docker tag ${AWS_ECR_REPO_NAME} ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER}'
                        sh 'docker push ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER}'
                }
            }
        }
        stage('Anchore Grype Vulnerability Scan') {
            steps {
                script {
                    sh 'docker run --rm anchore/grype:latest ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER} -o json > api-scan.json'
                    sh 'cat api-scan.json'
                }
            }
        }
    }
}
