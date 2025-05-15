// def  GIT_REPO_BRANCH
// GIT_REPO_BRANCH = env.GIT_BRANCH ? : 'develop'
// def GIT_REPO_URL = "https://github.com/AngelsTech/aws-lambda-demo"

pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git branch to build from')
        string(name: 'FUNCTION_NAME', defaultValue: 'my-lambda-function', description: 'Lambda Function Name')
        string(name: 'HANDLER_NAME', defaultValue: 'lambda_function.lambda_handler', description: 'Lambda Handler')
        string(name: 'ROLE_ARN', defaultValue: 'arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>', description: 'Lambda IAM Role ARN')
        string(name: 'SCHEDULE_EXPRESSION', defaultValue: 'rate(15 minutes)', description: 'CloudWatch Schedule Expression')
    }

    environment {
        REPO_URL = 'https://github.com/AngelsTech/aws-lambda-demo.git'
        REGION = 'us-east-1'
        RUNTIME = 'python3.13'
        ZIP_FILE = 'lambda.zip'
        ARN_FILE = 'lambda_arn.txt'
        SCHEDULE_RULE_NAME = 'lambda-schedule-rule'
    }

    stages {

        stage('Show Selected Params') {
            steps {
                echo "Branch: ${params.GIT_BRANCH}"
                echo "Function: ${params.FUNCTION_NAME}"
                echo "Handler: ${params.HANDLER_NAME}"
                echo "IAM Role ARN: ${params.ROLE_ARN}"
                echo "Schedule Expression: ${params.SCHEDULE_EXPRESSION}"
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: "${params.GIT_BRANCH}",
                    url: "${env.REPO_URL}",
                    credentialsId: '66142c87-d271-41f4-82d1-cbbee8e844d0'
            }
        }

        stage('Package Lambda Code') {
            steps {
                sh "zip -r ${env.ZIP_FILE} lambda_function.py"
            }
        }

        stage('Deploy Lambda Function') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'lambda-function-aws-cred',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh """
                        echo "Creating or updating Lambda function..."

                        aws lambda get-function \
                          --function-name ${params.FUNCTION_NAME} \
                          --region ${env.REGION} && EXISTS=true || EXISTS=false

                        if [ "\$EXISTS" = "false" ]; then
                            echo "Function doesn't exist. Creating..."
                            aws lambda create-function \
                              --function-name ${params.FUNCTION_NAME} \
                              --runtime ${env.RUNTIME} \
                              --role ${params.ROLE_ARN} \
                              --handler ${params.HANDLER_NAME} \
                              --zip-file fileb://${env.ZIP_FILE} \
                              --region ${env.REGION}
                        else
                            echo "Function exists. Updating code..."
                            aws lambda update-function-code \
                              --function-name ${params.FUNCTION_NAME} \
                              --zip-file fileb://${env.ZIP_FILE} \
                              --region ${env.REGION}
                        fi

                        echo "Getting Function ARN..."
                        aws lambda get-function \
                            --function-name ${params.FUNCTION_NAME} \
                            --region ${env.REGION} \
                            --query 'Configuration.FunctionArn' \
                            --output text > ${env.ARN_FILE}
                    """
                }
            }
        }

        stage('Setup CloudWatch Schedule') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'lambda-function-aws-cred',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh """
                        LAMBDA_ARN=\$(cat ${env.ARN_FILE})

                        echo "Creating/Updating CloudWatch rule..."
                        aws events put-rule \
                            --schedule-expression "${params.SCHEDULE_EXPRESSION}" \
                            --name ${env.SCHEDULE_RULE_NAME} \
                            --region ${env.REGION}

                        echo "Allowing CloudWatch to trigger Lambda..."
                        aws lambda add-permission \
                            --function-name ${params.FUNCTION_NAME} \
                            --statement-id schedule-invoke \
                            --action lambda:InvokeFunction \
                            --principal events.amazonaws.com \
                            --source-arn arn:aws:events:${env.REGION}:$(aws sts get-caller-identity --query Account --output text):rule/${env.SCHEDULE_RULE_NAME} \
                            --region ${env.REGION} || true

                        echo "Attaching rule to Lambda..."
                        aws events put-targets \
                            --rule ${env.SCHEDULE_RULE_NAME} \
                            --targets "Id"="1","Arn"="\${LAMBDA_ARN}" \
                            --region ${env.REGION}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Lambda deployment and schedule setup complete."
        }
        failure {
            echo "❌ Pipeline failed. Please check the logs above."
        }
    }
}
