pipeline {
    agent any

    tools {
        maven "Maven"
    }

    environment {
        AWS_ID = credentials("AWS-ACCOUNT-ID")
        REGION = "us-east-1"
        PROJECT = "transaction-microservice"
        COMMIT_HASH = "${sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()}"
    }

    stages {
        stage("Test") {
            steps {
                sh "git submodule init"
                sh "git submodule update"
                // sh "mvn clean test"
            }
        }

        stage("Package")  {
            steps {
                sh "mvn clean package -DskipTests"
            }
        }

        stage("SonarQube Code Analysis") {
            steps {
                withSonarQubeEnv(installationName: 'SonarQube-Server-kwi'){
                    sh "mvn sonar:sonar -Dsonar.projectName=${PROJECT}-kwi"
                }
            }
        }

        stage('Quality Gate'){
            steps {
                timeout(time: 3, unit: 'MINUTES'){
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Docker Build") {
            steps {
                sh "aws ecr get-login-password --region ${REGION} --profile keshaun | sudo docker login --username AWS --password-stdin ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com"
                sh "sudo docker build -t ${PROJECT}-kwi:${COMMIT_HASH} ."
                sh "sudo docker tag ${PROJECT}-kwi:${COMMIT_HASH} ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"
                sh "sudo docker push ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"
            }
        }
    }

    post {
        always {
            sh "sudo docker image rm ${PROJECT}-kwi:${COMMIT_HASH}"
            sh "sudo docker image rm ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"
            sh "mvn clean"
        }
    }
}