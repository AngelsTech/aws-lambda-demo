pipeline {
    agent any

    environment {
        FUNCTION_NAME = 'hey-world-first_lambdatest2'
        REGION = 'us-east-1'
        ZIP_FILE = "lambda_package.zip"
        ARN_FILE = 'lambda_arn.txt'
        ROLE_ARN = 'arn:aws:iam::529088259986:role/service-role/s3_execRole'  // added here for clarity
    }

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Enter git branch', trim: true)
    }

    stages {
        stage('Clone Repository') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: "*/${params.GIT_BRANCH}"]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/AngelsTech/aws-lambda-demo.git',
                        credentialsId: '66142c87-d271-41f4-82d1-cbbee8e844d0'
                    ]]
                ])
            }
        }

        stage('Prepare') {
            steps {
                sh '''
                    echo "Zipping all required files for Lambda deployment..."
                    # Zip everything at root except .git, README.md, Jenkinsfile
                    zip -r $ZIP_FILE . -x ".git/*" -x "README.md" -x "Jenkinsfile"
                '''
            }
        }

        stage('Deploy Lambda') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'lambda-function-aws-cred',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        echo "Creating Lambda function if it does not exist..."
                        aws lambda create-function \
                          --function-name $FUNCTION_NAME \
                          --runtime python3.13 \
                          --role $ROLE_ARN \

                          --handler lambda_function.lambda_handler \
                          --zip-file fileb://$ZIP_FILE \
                          --region $REGION || true

                        echo "Waiting for Lambda function to become Active..."
                        while true; do
                            STATUS=$(aws lambda get-function-configuration \
                                --function-name $FUNCTION_NAME \
                                --region $REGION \
                                --query 'State' --output text)
                            echo "Current status: $STATUS"
                            if [ "$STATUS" = "Active" ]; then
                                break
                            fi
                            sleep 5
                        done

                        echo "Updating function code..."
                        aws lambda update-function-code \
                            --function-name $FUNCTION_NAME \
                            --zip-file fileb://$ZIP_FILE \
                            --region $REGION

                        echo "Getting Lambda ARN..."
                        aws lambda get-function \
                            --function-name $FUNCTION_NAME \
                            --region $REGION \
                            --query 'Configuration.FunctionArn' \
                            --output text > $ARN_FILE
                    '''
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
                    sh '''
                        LAMBDA_ARN=$(cat $ARN_FILE)

                        echo "Creating CloudWatch Events rule..."
                        aws events put-rule \
                            --schedule-expression "rate(15 minutes)" \
                            --name hello-first-schedule \
                            --region $REGION

                        echo "Adding permission for CloudWatch to invoke Lambda..."
                        aws lambda add-permission \
                            --function-name $FUNCTION_NAME \
                            --statement-id hello-first-event \
                            --action lambda:InvokeFunction \
                            --principal events.amazonaws.com \
                            --region $REGION || true

                        echo "Creating target for CloudWatch Events..."
                        aws events put-targets \
                            --rule hello-first-schedule \


                            --targets "Id"="1","Arn"="$LAMBDA_ARN" \
                            --region $REGION
                    '''
                }
            }
        }
    }
}
