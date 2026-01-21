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
"""
    }
  }

  stages {
    stage('Run Python Script') {
      steps {
        container('python') {
          sh 'python scripts/run.py'
        }
      }
    }
  }
}
