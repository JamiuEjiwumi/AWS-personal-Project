#!/usr/bin/env groovy
import java.util.Date
import groovy.json.*
def repoName = 'sms-reporting-dashboard-frontend'
def projectName = 'sms-reporting-dashboard-frontend'
def deploymentName = 'sms-reporting-dashboard-frontend'
def isMaster = env.BRANCH_NAME == 'master'
def isStaging = env.BRANCH_NAME == 'staging'
def start = new Date()
def err = null
def jobInfo = "${env.JOB_NAME} ${env.BUILD_DISPLAY_NAME} \n${env.BUILD_URL}"
def imageTag = "${env.BUILD_NUMBER}"
String jobInfoShort = "${env.JOB_NAME} ${env.BUILD_DISPLAY_NAME}"
String buildStatus
String timeSpent
currentBuild.result = "SUCCESS"
try {
    node {
        deleteDir()
        stage('initializing'){
            slackSend (color: 'good', message: "Initializing build process for `${repoName}` ...")
        }
        stage ('Checkout') {
            checkout scm
        }
       stage("test deployment") {
         try {
             sh 'kubectl apply --validate=true --dry-run=true -f kubernetes/ --context i-040acc9d4cf316c92@k8scluster.us-west-2.eksctl.io'
           }
           catch (error) {
             sh "kubectl config use-context i-040acc9d4cf316c92@k8scluster.us-west-2.eksctl.io"
             slackSend (color: 'danger', message: ":disappointed: Build failed: ${jobInfo} ${timeSpent}")
             throw error
          }
       }
      stage ('Push Docker to AWS ECR') {
            sh "\$(aws ecr get-login --no-include-email --region ${AWS_ECR_REGION})"
            sh "eval \$(ssh-agent); ssh-add /var/lib/jenkins/.ssh/terra-bot; npm --prefix app install"
            sh "docker build -t ${AWS_ECR_ACCOUNT}/${projectName}/${repoName}:${imageTag} ."
            pushImage(repoName, projectName, imageTag)
            slackSend (color: 'good', message: "docker image on `${env.BRANCH_NAME}` branch in `${repoName}` pushed to AWS ECR")
        }
      if(isMaster || isStaging){
            stage ('Deploy to Kubernetes') {
                deploy(deploymentName, imageTag, projectName,repoName, isMaster)
                slackSend (color: 'good', message: ":fire: Nice work! `${repoName}` deployed to Kubernetes")
            }
      }
      stage('Clean up'){
                sh "docker rmi ${AWS_ECR_ACCOUNT}/${projectName}/${repoName}:${imageTag}"
         }
    }
} catch (caughtError) {
    err = caughtError
    currentBuild.result = "FAILURE"
} finally {
     timeSpent = "\nTime spent: ${timeDiff(start)}"
    if (err) {
        slackSend (color: 'danger', message: ":disappointed: Build failed: ${jobInfo} ${timeSpent} ${err}")
    } else {
        if (currentBuild.previousBuild == null) {
            buildStatus = 'First time build'
        } else if (currentBuild.previousBuild.result == 'SUCCESS') {
            buildStatus = 'Build complete'
        } else {
            buildStatus = 'Back to normal'
        }
        slackSend (color: 'good', message: "${jobInfo}: ${timeSpent}")
    }
}
def deploy(deploymentName, imageTag, projectName,repoName, isMaster){
    def namespace = isMaster ? "production" : "staging"
    sh("sed -i.bak 's|${AWS_ECR_ACCOUNT}/${projectName}/${repoName}:latest|${AWS_ECR_ACCOUNT}/${projectName}/${repoName}:${env.BUILD_NUMBER}|' ./kubernetes/sms-reporting-dashboard-frontend-deployment.yml")
    try{
        sh "kubectl apply --namespace=${namespace}  -f kubernetes/ --context i-040acc9d4cf316c92@k8scluster.us-west-2.eksctl.io"
    }
    catch (err) {
        slackSend (color: 'danger', message: ":disappointed: Build failed: ${jobInfo} ${err}")
    }
}
def pushImage(repoName, projectName, imageTag){
    try{
        sh "docker push ${AWS_ECR_ACCOUNT}/${projectName}/${repoName}:${imageTag}"
    }catch(e){
        sh "aws ecr create-repository --repository-name ${projectName}/${repoName} --region ${AWS_ECR_REGION}"
         sh "aws ecr set-repository-policy --repository-name ${projectName}/${repoName} --policy-text file://policy.json --region ${AWS_ECR_REGION}"
        sh "docker push ${AWS_ECR_ACCOUNT}/${projectName}/${repoName}:${imageTag}"
    }
}
def timeDiff(st) {
    def delta = (new Date()).getTime() - st.getTime()
    def seconds = delta.intdiv(1000) % 60
    def minutes = delta.intdiv(60 * 1000) % 60
    return "${minutes} min ${seconds} sec"
}
