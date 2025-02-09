pipeline {
    environment {
        // AWS_ACCESS_KEY_ID     = credentials('JenkinsAWSKey')
        // AWS_SECRET_ACCESS_KEY = credentials('JenkinsAWSKeySecret')
        TF_VAR_eb_app_name = 'eb-govcloudsample'
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
                    git branch: 'cf_testing',
                        credentialsId: 'luther-github-ssh',
                        url: 'git@github.com:cynerge-consulting/non-containerized-pipeline-tf.git'
                }
            }
        }
        stage('provision-infrastructure') {
            when {
                anyOf {
                    branch 'main'
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
                branch 'main'
            }
            agent {
                docker {
                    image 'cynergeconsulting/aws-cli:latest'
                    args '-u root'
                    alwaysPull true
                }
            }
            steps {
                sh '''
                    VERSION=$( date '+%F_%H:%M:%S' )
                    zip -r ${BUILD_TAG}.zip .
                    aws s3 cp ./$BUILD_TAG.zip s3://$ARTIFACT_BUCKET/$TF_VAR_eb_app_name/
                    aws elasticbeanstalk create-application-version --application-name $TF_VAR_eb_app_name --version-label v${BUILD_NUMBER}_${VERSION} --description="Built by Jenkins job $JOB_NAME" --source-bundle S3Bucket="$ARTIFACT_BUCKET",S3Key="$TF_VAR_eb_app_name/${BUILD_TAG}.zip" --region=us-gov-west-1
                    sleep 2
                    aws elasticbeanstalk update-environment --application-name $TF_VAR_eb_app_name --environment-name development --version-label v${BUILD_NUMBER}_${VERSION} --region=us-gov-west-1
                '''
            }
        }
        stage('staging-deploy') {
            when {
                branch 'staging'
            }
            steps {
                sh '''
                    VERSION=$( date '+%F_%H:%M:%S' )
                    zip -r ${BUILD_TAG}.zip .
                    aws s3 cp ./$BUILD_TAG.zip s3://$ARTIFACT_BUCKET/$TF_VAR_eb_app_name/
                    aws elasticbeanstalk create-application-version --application-name $TF_VAR_eb_app_name --version-label v${BUILD_NUMBER}_${VERSION} --description="Built by Jenkins job $JOB_NAME" --source-bundle S3Bucket="$ARTIFACT_BUCKET",S3Key="$TF_VAR_eb_app_name/${BUILD_TAG}.zip" --region=us-gov-west-1
                    sleep 2
                    aws elasticbeanstalk update-environment --application-name $TF_VAR_eb_app_name --environment-name staging --version-label v${BUILD_NUMBER}_${VERSION} --region=us-gov-west-1
                '''
            }
        }
        stage('prod-deploy') {
            when {
                branch 'prod'
            }
            steps {
                sh '''
                    VERSION=$( date '+%F_%H:%M:%S' )
                    zip -r ${BUILD_TAG}.zip .
                    aws s3 cp ./$BUILD_TAG.zip s3://$ARTIFACT_BUCKET/$TF_VAR_eb_app_name/
                    aws elasticbeanstalk create-application-version --application-name $TF_VAR_eb_app_name --version-label v${BUILD_NUMBER}_${VERSION} --description="Built by Jenkins job $JOB_NAME" --source-bundle S3Bucket="$ARTIFACT_BUCKET",S3Key="$TF_VAR_eb_app_name/${BUILD_TAG}.zip" --region=us-gov-west-1
                    sleep 2
                    aws elasticbeanstalk update-environment --application-name $TF_VAR_eb_app_name --environment-name production --version-label v${BUILD_NUMBER}_${VERSION} --region=us-gov-west-1
                '''
            }
        }
    }
}