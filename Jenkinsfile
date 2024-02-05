pipeline {
    agent {  label params.JENKINS_AGENT_LABEL }
    tools {
        nodejs 'node20.11'
    }
    environment {
        def BUILD_DATE =new Date().format("dd-MMM")
        def IMAGE_VERSION = "IMAGE_VERSION=${BULID_DATE}-${BUILD_NUMBER}" 
      }
    
    stages {
        
         stage('Defining environemnt speific variables') {
            steps{
                script{
                        /*withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'acr-credentials', usernameVariable: 'ACR_USERNAME', passwordVariable: 'ACR_PASSWORD']]) {*/
                    withCredentials([
                            [$class: 'UsernamePasswordMultiBinding', credentialsId: 'acr_credentials', usernameVariable: 'ACR_USERNAME', passwordVariable: 'ACR_PASSWORD'],
                            //[$class: 'UsernamePasswordMultiBinding', credentialsId: 'acr_credentials_staging', usernameVariable: 'ACR_USERNAME_STAGING', passwordVariable: 'ACR_PASSWORD_STAGING']
                        ])
                        {   
              
                    if (params.ENV_TYPE == 'production') {
                        ACR_REGISTRY = "ftcontainerregistryprod.azurecr.io"
                        REGISTRY_REPO = "ft-formio"
                        //RM_SLACK_CHANNEL = "ft-svc-deployments"
                        //RM_SLACK_TOKEN = "08db7e7a-e27d-4760-8295-06fb022bfe05"
                        BUILD_COMMAND = "npm install"
                        DOCKER_IMAGE_TAG = "${BUILD_DATE}-${BUILD_NUMBER}"
                        DOCKER_COMMAND = "docker login ${ACR_REGISTRY} --username ${ACR_USERNAME} --password ${ACR_PASSWORD}"
                        //TAG = "${ACR_REGISTRY}/${REGISTRY_REPO}:${DOCKER_IMAGE_TAG}"
                        TAG = "${DOCKER_IMAGE_TAG}"
                    } 
                    commitId = sh(script: "git rev-parse --short HEAD",returnStdout: true)
                    commitMsg = sh(script: """git rev-list --format=%B --max-count=1 ${commitId}""",returnStdout: true) 
                }
            }
        }}

        stage('Validate Branch Name') {
                when { expression { params.BRANCH_NAME != "master" } }
                steps {
                    script {
                        errorMessage = ""
                        branchName = params.BRANCH_NAME
                        print("Branch Name: ${BRANCH_NAME}")
                        if (!(branchName =~ '^(feature|release|hotfix|bugfix)/(FT)-[0-9]+')) {
                            errorMessage = """\
                                ***********************************************************************************

                                Branch Name ${BRANCH_NAME} doesn't follow the naming convention.

                                If you are unsure what the error means, please check the User Guide
                                https://reshamandi.atlassian.net/wiki/spaces/NINJAS/pages/989757441/Git+Branch+and+Commit+Message+Naming+Convention

                                ***********************************************************************************
                            """.stripIndent()
                            currentBuild.result = 'FAILURE'
                            error(errorMessage)
                        }
                    }
                }
        }       

        stage('Checkout') {
            steps {
                echo "Checking out $BRANCH_NAME"
                /*checkout([$class: 'GitSCM', branches: [[name: '${BRANCH_NAME}']], extensions: [], userRemoteConfigs: [[credentialsId: 'svc_acct_github', url: 'git@github.com:reshamandi/reshamandi-backend-platform.git']]])*/
                checkout([$class: 'GitSCM', branches: [[name: '${BRANCH_NAME}']], extensions: [], userRemoteConfigs: [[credentialsId: 'svc_acct_github', url: 'https://github.com/fintrasuite/formio.git']]])
                echo "Checkout is completed" 
            }
        }
        
       
        stage ('Docker Build and Push Image to ACR'){
            steps{
                parallel(
                    ftformioui: {
                        script{
                        if(params.BUILD_FLAG){
                        echo "Building docker image for Fintra formio"
                        sh """
                            docker version
                            #docker image prune --all
                            npm install
                            docker build --no-cache --tag ${ACR_REGISTRY}/${REGISTRY_REPO}:${DOCKER_IMAGE_TAG} .

                            echo "Executing Docker Commands"
                            set +x; ${DOCKER_COMMAND}&> /dev/null
                            docker push ${ACR_REGISTRY}/${REGISTRY_REPO}:${DOCKER_IMAGE_TAG}
                            #aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${ACR_REGISTRY}
                            """ 
                        echo "Building docker image for Fintra formio is completed"
                        }
                     }
                  } 
                ) 
            }
        }
        
        
        stage("Deploy Fintra formio") {
            steps{
                script{
                if(params.DEPLOY_FLAG){ 
                    echo "Triggering Spinnaker Pipeline for Deployment "
                     sh """
                            echo 'TAG=${TAG}' > build.properties
                        """  
                    echo "Please Check Spinnaker Pipeline for Deployment Progress"
                       }
                }
            }
        }    
        
        }
    post { 
           always { 
               emailext body: '''$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS:
               Check console output at $BUILD_URL to view the results.''', subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!', to: '${BUILD_USER_EMAIL}'
               archiveArtifacts artifacts: 'build.properties', onlyIfSuccessful: true
            }
 
          /*  success {
               slackSend channel: "${RM_SLACK_CHANNEL}", color: 'good', message: """
               *Service Name*: ${env.JOB_NAME}\n*Branch Name*: ${env.BRANCH_NAME}\n*Started by*: ${env.BUILD_USER_EMAIL}\n*Status*: "Success"\n*commit Id and Message*: ${commitMsg}\n*Build Job*: ${env.BUILD_URL}""", 
               tokenCredentialId: "${RM_SLACK_TOKEN}"
            }
     
            failure {
               slackSend channel: "${RM_SLACK_CHANNEL}", color: 'danger', message:"""
               *Service Name*: ${env.JOB_NAME}\n*Branch Name*: ${env.BRANCH_NAME}\n*Started by*: ${env.BUILD_USER_EMAIL}\n*Status*: "Failed"\n*Commit Id and Message*: ${commitMsg}\n*Build Job*: ${env.BUILD_URL}""",
               tokenCredentialId: "${RM_SLACK_TOKEN}"
            } */
    }   
}   