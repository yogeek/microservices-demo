pipeline {
  agent any
  stages {
    stage('kubeconfig') {
      steps {
        sh 'echo coucou'
      }
    }

    stage('Test Connection') {
      steps {
        sh 'echo coucou'
      }
    }

    stage('kube2iam ') {
      parallel {
        stage('kube2iam ') {
          steps {
            sh 'echo coucou'
          }
        }

        stage('storage-class') {
          steps {
            sh 'echo coucou'
          }
        }

        stage('kured') {
          steps {
            sh 'echo coucou'
          }
        }

        stage('filebeat') {
          steps {
            sh 'echo coucou'
          }
        }

        stage('metrics-server') {
          steps {
            sh 'echo coucou'
          }
        }

        stage('Velero bucket') {
          steps {
            sh 'echo coucou'
          }
        }

        stage('grafana operator') {
          steps {
            sh 'echo coucou'
          }
        }

        stage('jaeger operator') {
          steps {
            sh 'echo coucou'
          }
        }

      }
    }

  }
}