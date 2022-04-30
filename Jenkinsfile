pipeline {
  options {
    timeout(time: 180, unit: 'MINUTES')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '250', daysToKeepStr: '5'))
  }

  agent any

  environment {
    REGISTRY                 = credentials('REGISTRY_NAME')
    DOCKER_REGISTRY_PASSWORD = credentials('DOCKER_REGISTRY_PASSWORD')
    DOCKER_REGISTRY_USER     = credentials('DOCKER_REGISTRY_USERNAME')
    GITHUB_CREDS             = credentials('GITHUB_CREDS')
  }

  stages {
    stage('Get GIT_COMMIT') {
      steps {
        script {
          GIT_COMMIT = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
        }
      }
    }
    stage('docker-build') {
      options {
        timeout(time: 10, unit: 'MINUTES')
      }

      steps {
        sh '''#!/usr/bin/env bash
          echo "Shell Process ID: $$"
          docker login --username $DOCKER_REGISTRY_USER --password $DOCKER_REGISTRY_PASSWORD
          docker build --tag ${REGISTRY}/sampleapp:${BRANCH_NAME}-${GIT_COMMIT} .
          docker push ${REGISTRY}/sampleapp:${BRANCH_NAME}-${GIT_COMMIT}
        '''
      }
    }
    stage('Clone Helm Chart repo') {
        steps {
            dir('argo-cd') {
                git branch: 'main',
                credentialsId: 'GITHUB_CREDS',
                url: 'git@github.com:truongdinhtrong/argo-cd.git'
                sh '''#!/usr/bin/env bash
                    git config --global user.email "truongdinhtrongctim@gmail.com"
                    git config --global user.name "jenkins-ci"
                '''
            }
        }
    }
    stage('Deploy DEV') {
        when {
            branch 'dev'
        }
        steps {
            sshagent(['GITHUB_CREDS']) {
                sh '''#!/usr/bin/env bash
                  echo "Shell Process ID: $$"
                  # Replace Repository and tag
                  cd ./argo-cd/sampleapp
                  sed -r "s/^(\\s*repository\\s*:\\s*).*/\\1${REGISTRY}\\/sampleapp/" -i values-dev.yaml
                  sed -r "s/^(\\s*tag\\s*:\\s*).*/\\1${BRANCH_NAME}-${GIT_COMMIT}/" -i values-dev.yaml
                  git commit -am 'Publish new version' && git push --set-upstream origin main || echo 'no changes'
                '''
            }
        }
    }
    stage('Deploy PROD') {
        when {
            branch 'prod'
        }
        steps {
            sshagent(['GITHUB_CREDS']) {
                sh '''#!/usr/bin/env bash
                  echo "Shell Process ID: $$"
                  # Replace Repository and tag
                  cd ./argo-cd/sampleapp
                  sed -r "s/^(\\s*repository\\s*:\\s*).*/\\1${REGISTRY}\\/sampleapp/" -i values-prod.yaml
                  sed -r "s/^(\\s*tag\\s*:\\s*).*/\\1${BRANCH_NAME}-${GIT_COMMIT}/" -i values-prod.yaml
                  git commit -am 'Publish new version' && git push --set-upstream origin main || echo 'no changes'
                '''
            }
        }
    }
  }
  post {
    failure {
        sh '''#!/usr/bin/env bash
          echo "Shell Process ID: $$"
        '''
        }
    }
}
