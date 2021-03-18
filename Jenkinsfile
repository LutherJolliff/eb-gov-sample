pipeline {
    environment {
        // AWS_ACCESS_KEY_ID     = credentials('JenkinsAWSKey')
        // AWS_SECRET_ACCESS_KEY = credentials('JenkinsAWSKeySecret')
        TF_VAR_eb_app_name = 'eb-govcloudsample-windows'
        TF_VAR_role_arn = credentials('tf-role-arn')
        AWS_ACCESS_KEY_ID = credentials('tf_aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('tf_secret_access_key_id')
        ARTIFACT_BUCKET = 'elasticbeanstalk-us-gov-west-1-851887862617'
    }

    // agent {
    //     docker {
    //         image 'luther007/cynerge_images:latest'
    //         args '-u root'
    //     }
    // }
    agent {
        node {
            label 'master'
        }
    }

    stages {
        stage('dependencies') {
            steps {
                sh 'whoami'
                echo 'Installing...'
                // sh 'npm ci'
            }
        }
        stage('test') {
            steps {
                echo 'Testing...'
                // sh 'npm test'
            }
        }
        // stage('Build') {
        //     steps {
        //         script {
        //             dockerImage = docker.build registry + ":$BUILD_NUMBER"
        //         }
        //     }
        // }
        stage('clone-iaas-repo') {
            steps {
                sh 'rm terraform -rf; mkdir terraform'
                dir ('terraform') {
                    git branch: 'cf_testing_windows',
                        credentialsId: 'luther-github-ssh',
                        url: 'git@github.com:cynerge-consulting/non-containerized-pipeline-tf.git'
                }
            }
        }
        stage('provision-infrastructure') {
            when {
                anyOf {
                    branch 'main'
                    branch 'windows'
                }
            }
            steps {
                script {
                    sh 'ls'
                    dir ('terraform') {
                        sh '''
                            terraform --version
                            terraform init
                            terraform plan -input=false
                            terraform apply --auto-approve
                        '''
                    }
                }
            }
        }
        stage('dev-deploy') {
            when {
                branch 'windows'
            }
            // agent {
            //     docker {
            //         image 'cynergeconsulting/aws-cli:latest'
            //         args '-u root'
            //         alwaysPull true
            //     }
            // }
            steps {
                sh '''
                    VERSION=$( date '+%F_%H:%M:%S' )
                    docker run --rm -v $(pwd):/app -w /app mcr.microsoft.com/dotnet/sdk:5.0 dotnet publish -o site
                    zip -r site.zip site/*
                    zip ${BUILD_TAG}.zip site.zip aws-windows-deployment-manifest.json
                    ls -la
                    docker run --rm -v ~/.aws:/root/.aws -v $(pwd):/aws amazon/aws-cli:2.0.6 s3 cp ./${BUILD_TAG}.zip s3://$ARTIFACT_BUCKET/$TF_VAR_eb_app_name/
                    docker run --rm -v ~/.aws:/root/.aws -v $(pwd):/aws amazon/aws-cli:2.0.6 elasticbeanstalk create-application-version --application-name $TF_VAR_eb_app_name --version-label v${BUILD_NUMBER}_${VERSION} --description="Built by Jenkins job $JOB_NAME" --source-bundle S3Bucket="$ARTIFACT_BUCKET",S3Key="$TF_VAR_eb_app_name/${BUILD_TAG}.zip" --region=us-gov-west-1
                    sleep 2
                    docker run --rm -v ~/.aws:/root/.aws -v $(pwd):/aws amazon/aws-cli:2.0.6 elasticbeanstalk update-environment --application-name $TF_VAR_eb_app_name --environment-name development --version-label v${BUILD_NUMBER}_${VERSION} --region=us-gov-west-1
                '''
            }
        }
        stage('staging-deploy') {
            when {
                branch 'staging'
            }
            steps {
                sh '''
                    echo 'export PATH="/root/.ebcli-virtual-env/executables:$PATH"' >> ~/.bash_profile
                    eb deploy staging
                    sleep 5
                    aws elasticbeanstalk describe-environments --environment-names staging --query "Environments[*].CNAME" --output text
                '''
            }
        }
        stage('prod-deploy') {
            when {
                branch 'prod'
            }
            steps {
                sh '''
                    echo 'export PATH="/root/.ebcli-virtual-env/executables:$PATH"' >> ~/.bash_profile
                    eb deploy production
                    sleep 5
                    aws elasticbeanstalk describe-environments --environment-names production --query "Environments[*].CNAME" --output text
                '''
            }
        }
    }
}