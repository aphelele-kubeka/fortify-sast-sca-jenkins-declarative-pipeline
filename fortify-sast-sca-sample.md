# Jenkins Integration with Fortify SC SAST, Debricked and SSC

pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/aphelele-kubeka/springbootwebapp.git']]])
                sh "ls -lart ./*"
            }
        }
        stage('Fortify Remote Arguments') {
          tools {
            jdk 'JDK11'
          }
            steps {
                fortifyRemoteArguments transOptions: '-Xmx4G'
            }
        }
        stage('Fortify Remote Analysis') {
           tools {
            jdk 'JDK11'
          }
            steps {
                fortifyRemoteAnalysis remoteAnalysisProjectType: fortifyOther(),
                uploadSSC: [appName: 'SpringbootWebApp', appVersion: 'v1.0']
            }
        }
        stage('Debricked Scan') {
            steps {
                withCredentials([string(credentialsId: 'DEBRICKED_TOKEN', variable: 'DEBRICKED_TOKEN')]) {
                script {
                    sh 'curl -L https://github.com/debricked/cli/releases/latest/download/cli_linux_x86_64.tar.gz | tar -xz debricked'
                    sh './debricked scan'
                     }
 
                 }
             }
        }
        stage('Debricked Report Upload to SSC') {
            steps {
                withCredentials([string(credentialsId: 'DEBRICKED_TOKEN', variable: 'DEBRICKED_TOKEN')]) {
                script {
                    sh 'curl -L https://github.com/fortify/fcli/releases/download/v1.3.2/fcli-linux.tgz | tar -xz fcli'
                    sh './fcli ssc session login --insecure --socket-timeout=60s --connect-timeout=60s --url=http://localhost:8080/ssc --ci-token=$FORTIFY_CI_TOKEN'
                    sh './fcli ssc appversion-artifact import debricked --appversion=10029 --repository=mycontract/hierarchy-search-api --branch=master --debricked-access-token=$DEBRICKED_TOKEN'
                  }
                }
            }
        }
    }
}