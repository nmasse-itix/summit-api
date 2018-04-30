#!/usr/bin/env groovy
import groovy.json.JsonOutput

// for inline scripts
//def gitRepo = "https://github.com/nmasse-itix/summit-api.git"
//def gitBranch = "master"

// For Jenkinsfile from GIT
def gitRepo = scm.getUserRemoteConfigs()[0].getUrl()
def gitBranch = scm.getUserRemoteConfigs()[0].getRefspec()

node('nodejs') {
  def towerExtraVars = [
      git_repo: gitRepo,
      git_branch: gitBranch,
      threescale_cicd_api_backend_hostname: params.OPENSHIFT_SERVICE_NAME,
      threescale_cicd_api_backend_scheme: "http"
  ]

  // Get Source Code from SCM (Git) as configured in the Jenkins Project
  stage('Checkout Source') {
    // For Jenkinsfile from GIT
    checkout scm

    // for inline scripts
    //git url: gitRepo
  }

  def thisPackage = readJSON file: 'package.json'
  def currentVersion = thisPackage.version
  def newVersion = "$currentVersion-$BUILD_NUMBER"

  stage('Unit Tests') {
    sh "npm test"
  }

  // Build the OpenShift Image in OpenShift using the artifacts from NPM
  // Also tag the image
  stage('Build OpenShift Image') {
    // Trigger an OpenShift build in the dev environment
    openshiftBuild bldCfg: params.OPENSHIFT_BUILD_CONFIG, checkForTriggeredDeployments: 'false',
                   namespace: params.OPENSHIFT_BUILD_PROJECT, showBuildLogs: 'true',
                   verbose: 'false', waitTime: '', waitUnit: 'sec', env: [ ]


    // Tag the new build
    openshiftTag alias: 'false', destStream: params.OPENSHIFT_IMAGE_STREAM, destTag: "${newVersion}",
                 destinationNamespace: params.OPENSHIFT_BUILD_PROJECT, namespace: params.OPENSHIFT_BUILD_PROJECT,
                 srcStream: params.OPENSHIFT_IMAGE_STREAM, srcTag: 'latest', verbose: 'false'

  }
  stage('Deploy API to dev') {
    // Tag the new build as "ready-for-dev"
    openshiftTag alias: 'false', destStream: params.OPENSHIFT_IMAGE_STREAM, srcTag: "${newVersion}",
                 destinationNamespace: params.OPENSHIFT_DEV_ENVIRONMENT, namespace: params.OPENSHIFT_BUILD_PROJECT,
                 srcStream: params.OPENSHIFT_IMAGE_STREAM, destTag: 'ready-for-dev', verbose: 'false'

    // Trigger a new deployment
    openshiftDeploy deploymentConfig: params.OPENSHIFT_DEPLOYMENT_CONFIG, namespace: params.OPENSHIFT_DEV_ENVIRONMENT

    // Deploy the API to 3scale
    ansibleTower towerServer: params.ANSIBLE_TOWER_SERVER,
                 inventory: params.ANSIBLE_DEV_INVENTORY,
                 jobTemplate: params.ANSIBLE_JOB_TEMPLATE,
                 extraVars: JsonOutput.toJson(towerExtraVars)

  }
  stage('Deploy API to test') {
    // Tag the new build as "ready-for-test"
    openshiftTag alias: 'false', destStream: params.OPENSHIFT_IMAGE_STREAM, srcTag: "${newVersion}",
                 destinationNamespace: params.OPENSHIFT_TEST_ENVIRONMENT, namespace: params.OPENSHIFT_BUILD_PROJECT,
                 srcStream: params.OPENSHIFT_IMAGE_STREAM, destTag: 'ready-for-test', verbose: 'false'

    // Trigger a new deployment
    openshiftDeploy deploymentConfig: params.OPENSHIFT_DEPLOYMENT_CONFIG, namespace: params.OPENSHIFT_TEST_ENVIRONMENT

    // Deploy the API to 3scale
    ansibleTower towerServer: params.ANSIBLE_TOWER_SERVER,
                 inventory: params.ANSIBLE_TEST_INVENTORY,
                 jobTemplate: params.ANSIBLE_JOB_TEMPLATE,
                 extraVars: JsonOutput.toJson(towerExtraVars)

  }
  stage('Deploy API to prod') {
    // Tag the new build as "ready-for-prod"
    openshiftTag alias: 'false', destStream: params.OPENSHIFT_IMAGE_STREAM, srcTag: "${newVersion}",
                 destinationNamespace: params.OPENSHIFT_PROD_ENVIRONMENT, namespace: params.OPENSHIFT_BUILD_PROJECT,
                 srcStream: params.OPENSHIFT_IMAGE_STREAM, destTag: 'ready-for-prod', verbose: 'false'

    // Trigger a new deployment
    openshiftDeploy deploymentConfig: params.OPENSHIFT_DEPLOYMENT_CONFIG, namespace: params.OPENSHIFT_PROD_ENVIRONMENT

    // Deploy the API to 3scale
    ansibleTower towerServer: params.ANSIBLE_TOWER_SERVER,
                 inventory: params.ANSIBLE_PROD_INVENTORY,
                 jobTemplate: params.ANSIBLE_JOB_TEMPLATE,
                 extraVars: JsonOutput.toJson(towerExtraVars)
  }

}
