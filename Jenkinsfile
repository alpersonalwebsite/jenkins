pipeline {
  agent any
  stages {
    stage('Build') {
      steps {
        sh 'echo "Hello World"'
      }
    }

    stage('Lint HTML') {
      steps {
        sh 'tidy -q -e *.html'
      }
    }

    stage('Upload to AWS') {
      steps {
        withAWS(region: 'us-east-1', credentials: 'MyCredentials') {
          s3Upload(pathStyleAccessEnabled: true, payloadSigningEnabled: true, file: 'index.html', bucket: 'jenkins-test-123456')
        }

      }
    }

  }
}