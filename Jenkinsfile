pipeline {

  agent any
  options { timestamps() }
  
  environment {
    TF_DOCKER_IMAGE      = "teaas-tooling:${env.TEAAS_TOOLING_VERSION}"
    DOCKER_REGISTRY      = "${env.DEFAULT_ACR}"
    REGISTRY_CREDENTIALS = "DEFAULT_ACR_CREDENTIAL"
    AZ_DEVOPS_TOKEN      = "az-devops-token"
    ARM_USE_MSI          = true
  }
  
  stages{
    stage('checkout') {
      steps {
        checkout scm
      }
    }
    
    stage ('init plan apply') {
      steps {
        script {
          withEnv([
            "GIT_ASKPASS=${WORKSPACE}/askpass.sh",
            "ARM_SUBSCRIPTION_ID=${env.gbl_subscription_id}",]){
            withCredentials([usernamePassword(credentialsId: "${env.AZ_DEVOPS_TOKEN}", 
                                            passwordVariable: 'GIT_PASSWORD',
                                            usernameVariable: 'GIT_USERNAME')]){
              docker.withRegistry("${DOCKER_REGISTRY}", "${REGISTRY_CREDENTIALS}") {
                              // Pull the Docker image from the registry
                docker.image(TF_DOCKER_IMAGE).pull()
                docker.image(TF_DOCKER_IMAGE).inside() {
                  sh 'az login --identity'
                  sh 'az account set -s "${ARM_SUBSCRIPTION_ID}"'
                  for (stack in TF_STACK) {
                    def TF_EXEC_PATH = stack
                    def TF_BACKEND_CONF = "-backend-config='storage_account_name=${env.environment}empty' -backend-config='resource_group_name=cmn-${env.environment}-                     ${env.region_abbreviation}-gbltfstate-rg' -backend-config='key=${env.environment}/${env.vertical}/global-${env.region_abbreviation}/${stack}/terraform.tfstate'"
                    def TF_COMMAND = "terraform init ${TF_BACKEND_CONF}; terraform plan -var-file terraform.${env.environment}.${env.vertical}.${env.region_abbreviation}.tfvars -detailed-                     exitcode;"
                    def TF_COMMAND2 = "terraform apply -auto-approve -var-file terraform.${env.environment}.${env.vertical}.${env.region_abbreviation}.tfvars"
                    def exists = fileExists "${TF_EXEC_PATH}/terraform.${env.environment}.${env.vertical}.${env.region_abbreviation}.tfvars"
                    if (exists) {
                      def ret = sh(script: "cd ${TF_EXEC_PATH} && ${TF_COMMAND}", returnStatus: true)
                      println "TF plan exit code: ${ret}. \n INFO: 0 = Succeeded with empty diff (no changes);  1 = Error; 2 = Succeeded with non-empty diff (changes present)"
                      if ( "${ret}" == "0" ) {
                        sh "cd ${TF_EXEC_PATH} && ${TF_COMMAND2}"
                      }
                      else if ( "${ret}" == "1" ) {
                        error("Build failed because TF plan for ${stack} failed..")
                      }
                      else if ( "${ret}" == "2" ) {
                        if ( !params.autoApprove ) {
                          timeout(time: 1, unit: 'HOURS') {
                          input 'Approve the plan to proceed and apply'
                          }
                        }
                        sh "cd ${TF_EXEC_PATH} && ${TF_COMMAND2}"
                      }
                    }
                    else
                      echo " terraform.${env.environment}.${env.vertical}.${env.region_abbreviation}.tfvars doesn't exists in stack: ${stack} "
                  }
                }
              }
            }
          }
        }
      }
    }
    stage ('tag') {
      steps {
        withEnv([
          "GIT_ASKPASS=${WORKSPACE}/askpass.sh",
        ]) {
          withCredentials([usernamePassword(credentialsId: 'az-devops-token',
                                        passwordVariable: 'GIT_PASSWORD',
                                        usernameVariable: 'GIT_USERNAME')]) {
            script {
              env.TIMESTAMP = new SimpleDateFormat("yyyy-MM-dd-HH:mm:ss-Z").format(new Date())
            }
          gitTag(GIT_URL, "${env.environment}-${env.region_abbreviation}-live", "Deployed @ ${TIMESTAMP}", AZ_DEVOPS_TOKEN)
          }
        }
      }
    }
  }
  post {
    always {
      cleanWs()
    }
  }
}
