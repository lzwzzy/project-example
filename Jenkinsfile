pipeline {
  agent any
  stages {
    stage('SCM') {
      steps {
        git(url: 'https://github.com/JFrogChina/project-example.git', branch: 'master', changelog: true, credentialsId: 'my-git-hub', poll: true)
      }
    }
  }
  environment {
    mvnHome = ''
    artiServer = ''
    rtMaven = ''
    buildInfo = ''
  }
}