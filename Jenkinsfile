pipeline {
  agent any
  stages {
    stage('test') {
      parallel {
        stage('test') {
          steps {
            echo 'hello'
          }
        }

        stage('') {
          steps {
            echo 'hello'
          }
        }

      }
    }

  }
  environment {
    key = '1'
  }
}