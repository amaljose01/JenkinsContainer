pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: runner
    image: ubuntu:22.04
    command:
    - cat
    tty: true
"""
    }
  }

  stages {
    stage('Checkout') {
      steps {
        container('runner') {
          checkout scm
        }
      }
    }

    stage('Run scripts') {
      steps {
        container('runner') {
          sh '''
            chmod +x scripts/*.sh
            scripts/script1.sh
            scripts/script2.sh
          '''
        }
      }
    }
  }
}
