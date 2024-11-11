pipeline {
    agent { label 'jenkins-Agent' }
    tools {
        jdk 'java17'
        maven 'maven3'
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "ajs23" // Replace with your Docker Hub username
        DOCKER_PASS = credentials('dockerhub') // Use the credentials ID for Docker Hub
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }

    stages {
        stage('Cleanup Workspace') {
            steps {
                echo 'Cleaning workspace'
                cleanWs() // Optional: Cleans the workspace
            }
        }

        stage('Checkout from SCM') {
            steps {
                echo 'Retrieving the source code from GitHub'
                git branch: 'main', credentialsId: 'Github', url: 'https://github.com/AJS23/register-app.git'
            }
        }

        stage('Application Build') {
            steps {
                echo 'Hurray, source code fetched, moving to further steps and building the application'
                sh 'mvn clean package'
            }
        }

        stage('Application Test') {
            steps {
                echo 'Hurray, application built successfully, moving to further steps and testing the application'
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube analysis'
                script {
                    withSonarQubeEnv('sonarqube') { 
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'We are in the quality gate now'
                script {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                        def docker_image = docker.build("${IMAGE_NAME}")
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    sh 'docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ashfaque9x/register-app-pipeline:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table'
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Trigger CD Pipeline') {
            steps {
                script {
                    sh "curl -v -k --user jenkins:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-65-2-80-33.ap-south-1.compute.amazonaws.com:8080/job/argocd/buildWithParameters?token=gitops-token'"
                }
            }
        }
    }

    post {
        failure {
            emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                      subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed",
                      mimeType: 'text/html', to: "ashfaque.s510@gmail.com"
        }
        success {
            emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                      subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful",
                      mimeType: 'text/html', to: "ashfaque.s510@gmail.com"
        }
    }
}
