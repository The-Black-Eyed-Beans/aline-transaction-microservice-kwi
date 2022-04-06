pipeline {
    agent {
        node {
            label "worker-one"
        }
    }

    tools {
        maven "Maven"
    }

    environment {
        AWS_ID = credentials("AWS-ACCOUNT-ID")
        REGION = credentials("REGION-KWI")
        APP = "transaction"
        PROJECT = "transaction-microservice"
        COMMIT_HASH = "${sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()}"
        APP_PORT = 8073
        DEPLOYMENT = "EKS"
    }

    stages {
        stage("Test") {
            steps {
                sh "git submodule init"
                sh "git submodule update"
                // sh "mvn clean test"
            }
        }

        stage("SonarQube Code Analysis") {
            steps {
                withSonarQubeEnv(installationName: 'SonarQube-Server-kwi'){
                    sh "mvn verify -Dmaven.test.failure.ignore=true sonar:sonar -Dsonar.projectName=${PROJECT}-kwi"
                }
            }
        }

        stage('Quality Gate'){
            steps {
                timeout(time: 5, unit: 'MINUTES'){
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Package")  {
            steps {
                sh "mvn clean package -DskipTests"
            }
        }

        stage("Docker Build") {
            steps {
                echo "Authenticating with AWS Credentials..."
                sh "docker context use default"
                sh "aws ecr get-login-password --region ${REGION} --profile keshaun | docker login --username AWS --password-stdin ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com"

                echo "Building Docker Image with Commit Hash as the tag..."
                sh "docker build -t ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH} ."
                sh "docker push ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"

                echo "Building Docker Image with latest tag..."
                sh "docker tag ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH} ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:latest"
                sh "docker push ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:latest"
            }
        }

        stage("ECS Deployment") {
            when {
                environment(name: "DEPLOYMENT", value: "ECS")
            }
            steps {
                echo "Deploying ${PROJECT}-kwi..."
                sh '''
                aws cloudformation deploy \
                --stack-name ${PROJECT}-kwi-stack \
                --template-file deploy.json \
                --profile keshaun \
                --capabilities CAPABILITY_IAM \
                --no-fail-on-empty-changeset \
                --parameter-overrides \
                    MicroserviceName=${PROJECT} \
                    AppPort=${APP_PORT} \
                    ImageTag=${COMMIT_HASH}
                '''
            }
        }

        stage("EKS Deployment") {
            when {
                environment(name: "DEPLOYMENT", value: "EKS")
            }
            steps {
                echo "Generating .env..."
                sh """aws secretsmanager get-secret-value --secret-id aline-kwi/dev/secrets/resources --region us-east-1 --profile keshaun | jq -r '.["SecretString"]' | jq '.' | jq -r 'keys[] as \$k | "export \\(\$k)=\\(.[\$k])"' > .env"""
                sh """aws secretsmanager get-secret-value --secret-id aline-kwi/dev/secrets/user-credentials --region us-east-1 --profile keshaun | jq -r '.["SecretString"]' | jq '.' | jq -r 'keys[] as \$k | "export \\(\$k)=\\(.[\$k])"' >> .env"""
                sh """aws secretsmanager get-secret-value --secret-id aline-kwi/dev/secrets/db --region us-east-1 --profile keshaun | jq -r '.["SecretString"]' | jq '.' | jq -r 'keys[] as \$k | "export Db\\(\$k)=\\(.[\$k])"' >> .env"""
                
                sh "echo 'export ImageTag=${COMMIT_HASH}' >> .env"
                sh "echo 'export AppPort=${APP_PORT}' >> .env"
                sh "echo 'export AppName=${APP}' >> .env"
                sh "echo 'export Project=${PROJECT}' >> .env"
                sh "echo 'export AwsId=${AWS_ID}' >> .env"
                sh "echo 'export AwsRegion=${REGION}' >> .env"

                echo "Deploying ${PROJECT}-kwi..."
                sh "aws eks update-kubeconfig --name=aline-kwi-eks --region=us-east-1 --profile keshaun"
                sh ". ./.env && envsubst < deployment.yml | kubectl apply -f -"

                echo "Deleting .env..."
                sh "rm .env"
            }
        }
    }

    post {
        always {
            sh "docker image rm ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"
            sh "docker image rm ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:latest"
            sh "mvn clean"
        }
    }
}