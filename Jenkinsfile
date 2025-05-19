// def  GIT_REPO_BRANCH
// GIT_REPO_BRANCH = env.GIT_BRANCH ? : 'develop'
// def GIT_REPO_URL = "https://github.com/AngelsTech/aws-lambda-demo"

pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'develop', description: 'Git branch to build from')
        string(name: 'FUNCTION_NAME', defaultValue: 'lambda-function-testdemo', description: 'Lambda Function Name')
        string(name: 'HANDLER_NAME', defaultValue: 'lambda_function.lambda_handler', description: 'Lambda Handler')
        string(name: 'SCHEDULE_EXPRESSION', defaultValue: 'rate(15 minutes)', description: 'CloudWatch Schedule Expression')
    }

    environment {
        REPO_URL = 'https://github.com/AngelsTech/aws-lambda-demo.git'
        REGION = 'us-east-1'
        RUNTIME = 'python3.13'
        ZIP_FILE = 'lambda.zip'
        ARN_FILE = 'lambda_arn.txt'
        SCHEDULE_RULE_NAME = 'lambda-schedule-rule'
        LAMBDA_ROLE_ARN = 'arn:aws:iam::529088259986:role/service-role/s3_execRole' // Replace with your real ARN
    }

    stages {

        stage('Show Selected Params') {
            steps {
                echo "🔧 Branch: ${params.GIT_BRANCH}"
                echo "🔧 Function: ${params.FUNCTION_NAME}"
                echo "🔧 Handler: ${params.HANDLER_NAME}"
                echo "🔧 Schedule Expression: ${params.SCHEDULE_EXPRESSION}"
                echo "🔧 Lambda Role ARN: ${env.LAMBDA_ROLE_ARN}"
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
                sh """
                    set -e
                    echo "📦 Zipping lambda_function.py..."
                    zip -r ${env.ZIP_FILE} lambda_function.py
                """
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
                        set -e
                        echo "🚀 Checking if Lambda function exists..."

                        if aws lambda get-function --function-name ${params.FUNCTION_NAME} --region ${env.REGION}; then
                            echo "🔄 Function exists. Updating code..."
                            aws lambda update-function-code \\
                                --function-name ${params.FUNCTION_NAME} \\
                                --zip-file fileb://${env.ZIP_FILE} \\
                                --region ${env.REGION}
                        else
                            echo "🆕 Function doesn't exist. Creating..."
                            aws lambda create-function \\
                                --function-name ${params.FUNCTION_NAME} \\
                                --runtime ${env.RUNTIME} \\
                                --role ${env.LAMBDA_ROLE_ARN} \\
                                --handler ${params.HANDLER_NAME} \\
                                --zip-file fileb://${env.ZIP_FILE} \\
                                --region ${env.REGION}
                        fi

                        echo "📥 Fetching Function ARN..."
                        aws lambda get-function \\
                            --function-name ${params.FUNCTION_NAME} \\
                            --region ${env.REGION} \\
                            --query 'Configuration.FunctionArn' \\
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
                        set -e
                        LAMBDA_ARN=\$(cat ${env.ARN_FILE})
                        ACCOUNT_ID=\$(aws sts get-caller-identity --query Account --output text)

                        echo "⏰ Creating or updating CloudWatch rule..."
                        aws events put-rule \\
                            --schedule-expression "${params.SCHEDULE_EXPRESSION}" \\
                            --name ${env.SCHEDULE_RULE_NAME} \\
                            --region ${env.REGION}

                        echo "🔐 Adding permission to Lambda..."
                        aws lambda add-permission \\
                            --function-name ${params.FUNCTION_NAME} \\
                            --statement-id schedule-invoke \\
                            --action lambda:InvokeFunction \\
                            --principal events.amazonaws.com \\
                            --source-arn arn:aws:events:${env.REGION}:\${ACCOUNT_ID}:rule/${env.SCHEDULE_RULE_NAME} \\
                            --region ${env.REGION} || true

                        echo "🔗 Linking rule to Lambda target..."
                        aws events put-targets \\
                            --rule ${env.SCHEDULE_RULE_NAME} \\
                            --targets '[{"Id":"1","Arn":"'\${LAMBDA_ARN}'"}]' \\
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
