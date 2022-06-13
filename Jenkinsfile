pipeline {
  agent { label "${AGENT_LABEL}" }

  environment {
    DOCKERHUB_CREDENTIAL = credentials('docker-hub-credentials')
  }
  parameters {
      gitParameter (  branch: '', 
                      branchFilter: 'origin/(.*)', 
                      defaultValue: 'develop', 
                      description: '', 
                      name: 'BRANCH', 
                      quickFilterEnabled: true, 
                      selectedValue: 'TOP', 
                      sortMode: 'DESCENDING', 
                      tagFilter: '*', 
                      type: 'PT_BRANCH', 
                      useRepository: 'git@github.com:vhryshchenko-source/k8-practice.git')

      choice (  name: 'AGENT_LABEL', 
                choices: ['jenkins-slave-1', 'jenkins-slave-2'], 
                description: 'Choose jenkins worker where job will be run')
        
      choice (  name: 'PYTHON_VERSION', 
                choices: ['3.7', '3.8'], 
                description: 'Choose python version for docker base image')

      string (  name: 'DOCKER_REPO', 
                defaultValue: 'vhrysh/hit-count', 
                description: 'Docker hub repository name')

      string (  name: 'RELEASE_TAG', 
                description: 'Enter release tag if you want to make realese')

      string (  name: 'COMMIT_ID', 
                description: 'Enter git commit')
            
      choice (  name: 'BUILD_RELEASE', 
                choices: ['FALSE', 'TRUE'], 
                description: 'Choose TRUE if you want build and deploy image with release tag')
  }


    stages {
        stage('Checkout branch') {
            steps{
                // Checkout branch
                git branch: "${params.BRANCH}", credentialsId: 'github-credentials', url: 'git@github.com:vhryshchenko-source/k8-practice.git'
            }
        }
        stage('Checkout git commit') {
            when {
              expression {
                params.COMMIT_ID != ''
              }
            }
            steps{
                // Checkout commit
                checkout(
                    [$class: 'GitSCM', 
                    branches: [[name: "${params.COMMIT_ID}"]],
                    doGenerateSubmoduleConfigurations: false, 
                    extensions: [],
                    submoduleCfg: [], userRemoteConfigs: 
                    [[credentialsId: 'github-credentials', 
                    url: 'git@github.com:vhryshchenko-source/k8-practice.git']]
                    ]
                )
            }
        }
        stage('SonarQube analysis') {
          def scannerHome = tool 'SonarQube-scanner-4.7';
          withSonarQubeEnv('sonarqube') {
            sh "${scannerHome}/bin/sonar-scanner \
            -D sonar.projectName=test \
            -D sonar.projectKey=test \
            -D sonar.sources=."
          }
        }
        stage('Quality Gates'){
            
          timeout(time: 1, unit: 'HOURS') {
          def qg = waitForQualityGate() 
          if (qg.status != 'OK') {
            error "Pipeline aborted due to quality gate failure: ${qg.status}"
            }
          }
        }
        stage('Build image') {
          when {
            expression {
              BRANCH == 'develop' || BUILD_RELEASE == 'TRUE'
            }
          }
          steps {
            script {
                if ("${params.COMMIT_ID}" == '') {
                    container('docker') {
                    sh '''
                        docker build --tag $DOCKER_REPO:$GIT_COMMIT --build-arg PYTHON_VERSION .
                    '''
                    }
                } else {
                    container('docker') {
                    sh '''
                        docker build --tag $DOCKER_REPO:$RELEASE_TAG --build-arg PYTHON_VERSION .
                    '''
                    }
                }
            }
          }
        }
        stage('Docker hub login') {
          steps{
            container('docker') {
              sh 'echo $DOCKERHUB_CREDENTIAL_PSW | docker login -u $DOCKERHUB_CREDENTIAL_USR --password-stdin'
            }   
          }
        }
        stage('Pull image') {
          when {
            expression { 
              BRANCH != 'develop' && BUILD_RELEASE == 'FALSE'
            }
          }
          environment {
              GIT_COMMIT = "${sh(script:'git rev-parse --verify HEAD', returnStdout: true).trim()}"
          }
          steps{
              script {
                  if (params.COMMIT_ID != '' ) {
                    container('docker') {
                        sh 'docker pull $DOCKER_REPO:$COMMIT_ID'
                    }
                  }
                  else {
                    container('docker') {
                      sh 'docker pull $DOCKER_REPO:$GIT_COMMIT'
                    }
                  }
              }
          }
        }
        stage('Push image from develop branch') {
          when {
            expression { 
              BRANCH == 'develop' && params.COMMIT_ID == ''
            }
          }  
          steps{
            container('docker') {
                sh 'docker push $DOCKER_REPO:$GIT_COMMIT'
            }
          }
        }
        stage('Push image from release branch') {
          when {
            expression { 
              BRANCH != 'develop'
            }
          }
          environment {
              GIT_COMMIT = "${sh(script:'git rev-parse --verify HEAD', returnStdout: true).trim()}"
          } 
            steps {
                script {
                  if (BUILD_RELEASE == 'TRUE' && params.COMMIT_ID != '' ) {
                    container('docker') {
                        sh 'docker push $DOCKER_REPO:$RELEASE_TAG'
                    }
                  }
                  if (BUILD_RELEASE == 'FALSE' && params.COMMIT_ID != '' ) {
                    container('docker') {
                        sh 'echo PUSH by secific commit without build'
                        sh 'docker tag $DOCKER_REPO:$COMMIT_ID $DOCKER_REPO:$RELEASE_TAG'
                        sh 'docker push $DOCKER_REPO:$RELEASE_TAG'
                    }
                  }
                  if (BUILD_RELEASE == 'FALSE' && params.COMMIT_ID == '' ) {
                    container('docker') {
                      sh 'echo push with build in release branch'
                      sh 'docker tag $DOCKER_REPO:$GIT_COMMIT $DOCKER_REPO:$RELEASE_TAG'
                      sh 'docker push $DOCKER_REPO:$RELEASE_TAG'
                    }
                  }
                }
            }
        }
    }
    post {
        failure {
            slackSend failOnError: true, message: "Build failed  - ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
        }
    }
}
