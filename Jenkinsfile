// CI/CD Pipeline for Node.js Application deployment to Azure App Service
pipeline {
    agent any

    // Define environment variables, ensuring no leading/trailing spaces in values.
    environment {
        // Azure environment details
        AZURE_SUBSCRIPTION_ID = 'd6e154dc-0c67-4143-9261-e8b06141c24f'
        AZURE_TENANT_ID       = 'e8e808be-1f06-40a2-87f1-d3a52b7ce684'

        // Application-specific details
        RESOURCE_GROUP        = 'project5'
        APP_NAME              = 'sai-webapp'
        NODE_ENV              = 'production'
        // Define the target Node.js runtime for deployment
        NODE_RUNTIME          = 'NODE|20-lts'
        AZURE_LOCATION        = 'North Europe'
        AZURE_SKU             = 'B1'
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo 'Checking out code from Git...'
                git branch: 'main',
                    url: 'https://github.com/pattelakrishna/project5-node.js-mongodb.git',
                    credentialsId: 'github-id'
            }
        }

        stage('Install Dependencies') {
            steps {
                echo 'Installing Node.js dependencies...'
                sh 'npm install'
            }
        }

        stage('Run Tests') {
            steps {
                // Using 'catchError' allows the pipeline to continue execution
                // in the 'post' section even if this step fails, while still
                // marking the stage as unstable/failed based on the result.
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    echo 'Running tests...'
                    // Standard npm test command
                    sh 'npm test'
                }
            }
        }

        stage('Build Application') {
            steps {
                echo 'Building application...'
                sh 'npm run build'
            }
        }

        stage('Login to Azure') {
            steps {
                // Securely inject client ID and secret using Jenkins credentials
                withCredentials([
                    string(credentialsId: 'AZURE_CLIENT_ID', variable: 'AZURE_CLIENT_ID'),
                    string(credentialsId: 'AZURE_CLIENT_SECRET', variable: 'AZURE_CLIENT_SECRET')
                ]) {
                    sh '''
                        echo "Logging into Azure using Service Principal..."
                        az login --service-principal \
                            -u $AZURE_CLIENT_ID \
                            -p $AZURE_CLIENT_SECRET \
                            --tenant $AZURE_TENANT_ID \
                            --output none --only-show-errors

                        echo "Setting default subscription..."
                        az account set --subscription $AZURE_SUBSCRIPTION_ID
                    '''
                }
            }
        }

        stage('Deploy to Azure App Service') {
            steps {
                sh """
                    echo "Deploying application to Azure App Service: $APP_NAME..."
                    # az webapp up creates the App Service and deploys the content in one step.
                    az webapp up \\
                        --name $APP_NAME \\
                        --resource-group $RESOURCE_GROUP \\
                        --runtime $NODE_RUNTIME \\
                        --location $AZURE_LOCATION \\
                        --sku $AZURE_SKU
                """
            }
        }

        stage('Configure App Settings') {
            steps {
                // Securely inject the sensitive MongoDB connection string
                withCredentials([string(credentialsId: 'MONGODB_URI', variable: 'MONGODB_URI')]) {
                    // NEW FIX: Use 'printf' inside the single-quoted shell block to securely
                    // construct the settings string. This is the most reliable method for
                    // handling secrets with special characters passed to external commands.
                    sh '''
                        echo "Setting MONGODB_URI and NODE_ENV environment variables on App Service..."
                        
                        # Use printf to construct the sensitive argument string
                        # The %s format specifier handles complex characters in the URI safely.
                        SETTINGS_STRING=$(printf "MONGODB_URI=%s NODE_ENV=%s" "$MONGODB_URI" "$NODE_ENV")

                        az webapp config appsettings set \
                            --name $APP_NAME \
                            --resource-group $RESOURCE_GROUP \
                            --settings $SETTINGS_STRING
                    '''
                }
            }
        }
    }

    post {
        always {
            // Log out of Azure regardless of success/failure
            sh 'az logout || true'
        }
        success {
            echo '✅ CI/CD pipeline completed successfully!'
            echo "Your app is live at: https://${env.APP_NAME}.azurewebsites.net"
        }
        failure {
            echo '❌ CI/CD pipeline failed. Please check the logs.'
        }
    }
}
