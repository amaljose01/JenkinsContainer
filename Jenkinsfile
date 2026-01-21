pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: python
    image: ci/python-runner:1.0
    command: ['cat']
    tty: true

  - name: node
    image: ci/node-runner:1.0
    command: ['cat']
    tty: true
"""
    }
  }

  stages {
    stage('Build') {
      steps {
        container('node') {
          sh 'npm install'
        }
        container('python') {
          sh 'python scripts/run.py'
        }
      }
    }
  }
}
