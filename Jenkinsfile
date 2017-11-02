#!/usr/bin/env groovy

def projectProperties = [
  [$class: 'BuildDiscarderProperty',strategy: [$class: 'LogRotator', numToKeepStr: '5']],disableConcurrentBuilds(),
  [$class: 'ParametersDefinitionProperty', parameterDefinitions: [[$class: 'StringParameterDefinition', defaultValue: 'io/qa-staging', description: '', name: 'SITE_BRANCH']]]
]

properties(projectProperties)

try {
  node('docker') {
    // Cleanup within the container as we run as root
    docker.withRegistry('https://docker-registry.cloud.aws.tenablesecurity.com:8888/') {
      docker.image('ci-vulnautomation-base:1.0.9').inside("-u root") {
        stage('clean auto') {
          sh 'chown -R 1000:1000 .'
        }
      }
    } 

    deleteDir()

    // Pull the automation framework from develop
    stage('scm auto') {
      dir("tenableio-sdk") {
        checkout scm
      }
      dir("automation") {
        git branch: 'develop', changelog: false, credentialsId: 'bitbucket-checkout', poll: false, url: 'ssh://git@stash.corp.tenablesecurity.com:7999/aut/automation-tenableio.git'
      }
      dir("site") {
        git branch: "${params.SITE_BRANCH}", changelog: false, credentialsId: 'bitbucket-checkout', poll: false, url: 'ssh://git@stash.corp.tenablesecurity.com:7999/aut/site-configs.git'
      }
    }

    docker.withRegistry('https://docker-registry.cloud.aws.tenablesecurity.com:8888/') {
      docker.image('ci-vulnautomation-base:1.0.9').inside("-u root") {
        stage('build auto') {
          try {
            timeout(time: 60, unit: 'MINUTES') {
              sshagent(['buildenginer_public']) {
                sh 'git config --global user.name "buildenginer"'
                sh 'mkdir ~/.ssh && chmod 600 ~/.ssh'
                sh 'ssh-keyscan -H -p 7999 stash.corp.tenablesecurity.com >> ~/.ssh/known_hosts'
                sh 'ssh-keyscan -H -p 7999 172.25.100.131 >> ~/.ssh/known_hosts'
                sh '''
cd automation || exit 1
python3 autosetup.py catium --all --no-venv 2>&1
export PYTHONHASHSEED=0 
export PYTHONPATH=. 
export CAT_USE_GRID=true

python3 tenableio/commandline/sdk_test_container.py --create_container --python

cd ../tenableio-sdk || exit 1
pip3 install -r requirements.txt || exit 1
py.test tests --junitxml=test-results-junit.xml || exit 1
'''
              }
            }
          }
          finally {
            sh 'find . -name \*.xml'
	    step([$class: 'JUnitResultArchiver', testResults: 'tenableio-sdk/*.xml'])
          }
        }
      }
    } 
  }
}
catch (exc) {
  echo "caught exception: ${exc}"
  currentBuild.result = 'FAILURE'
}
