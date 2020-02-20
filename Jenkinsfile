UNIQUE_VERSION = UUID.randomUUID().toString()
K8S_TOOLSET_IMAGE = 'cloud/continental/builds/k8s-toolset:1.16'
AWS_MANAGER_IMAGE = 'cloud/continental/tools/aws-manager:1.3.1'
VELERO_IMAGE = 'cloud/continental/builds/velero:1.2.0'

pipeline {
  agent { label 'default' }

  parameters {
    choice(name: 'REGION', choices: ['eu-central-1', 'ap-northeast-2', 'us-east-1'], description: 'Target region')
    choice(name: 'WORK', choices: ['dev', 'test', 'prod'], description: 'Target workspace')
    choice(name: 'ENVIRONMENT', choices: ['integration', 'validation', 'production'], description: 'Target environment')
  }

  options {
    timeout(time: 20, unit: 'MINUTES')
    disableConcurrentBuilds()
    ansiColor('xterm')
  }

  stages {

    // K8S cluster
    stage('Get K8S kubeconfig') {
      agent {
        docker {
          image "${REPO_CI_PULL}/mesosphere/aws-cli"
          args "-u root --entrypoint=''"
          reuseNode true
          alwaysPull true
          registryUrl "https://${REPO_CI_PULL}"
          registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
        }
      }
      environment {
        AWS_CREDENTIALS_ID = 'aws-SA_gtp_deployment'
      }
      steps {
        script {
          currentBuild.displayName = "${BUILD_DISPLAY_NAME}-${params.ENVIRONMENT}-${params.WORK}"
        }
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CREDENTIALS_ID]]) {
          sh """
              target_s3_path=${KUBECONFIG_S3_BUCKET}/${params.ENVIRONMENT}/${params.REGION}
              if [ "${params.WORK}" != "prod" ]
              then
                target_s3_path=${KUBECONFIG_S3_BUCKET}/${params.WORK}/${params.ENVIRONMENT}/${params.REGION}
              fi
              aws --region ${params.REGION} s3 cp s3://\${target_s3_path}/kubeconfig kubeconfig.conf
            """
        }
      }
    }

    // K8S cluster
    stage('Test connection K8S') {
      when { branch 'master' }
      agent {
        docker {
          image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
          reuseNode true
          alwaysPull true
          registryUrl "https://${REPO_CI_PULL}"
          registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
        }
      }
      environment { 
        KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
      }
      steps {
          sh "kubectl cluster-info"
      }
    }

    

    stage('Parallel 1') {
      when { branch "master" }
      parallel {
        
        // Install Kube2iam
        stage('Install Kube2iam') {
          when { branch "master" }
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd kube2iam
                source install.sh
                source test.sh
              """
          }
        }

        // Install EBS storage class
        stage('Install Storage Class') {
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd storage-class
                source install.sh
                source test.sh
              """
          }
        }

        // Install Cert Manager
        stage('Install Cert Manager') {
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd cert-manager
                source install.sh
                source test.sh
              """
          }
        }
        
        // Install Metrics Server
        stage('Install Metrics Server') {
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd metrics-server
                source install.sh
                source test.sh
              """
          }
        }
        
        // Install Kured
        stage('Install Kured') {
          when { branch "master" }
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd kured
                source install.sh
              """
          }
        }

        // Install filebeat (only on main environments)
        stage('Install Filebeat') {
          when { 
              branch 'master'
              anyOf {
                  expression { params.ENVIRONMENT == 'integration' }
                  expression { params.ENVIRONMENT == 'validation' }
                  expression { params.ENVIRONMENT == 'production' }
              }
          }
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
              args "-u root"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
            AWS_DEFAULT_REGION = 'eu-central-1'
          }
          steps {
            sh """
              cd filebeat
              source install.sh ${params.ENVIRONMENT}
            """
          }
        }

        // Create Velero S3 Bucket for backups (TO MOVE IN CLUSTER CREATION REPO ?)
        stage('Create Velero S3 bucket for backups') {
          when { branch "master" }
          agent {
            docker {
                image "${AWS_MANAGER_IMAGE}"
                alwaysPull true
                reuseNode true
                registryUrl "https://${REPO_CI_PULL}"
                registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
            AWS_DEFAULT_REGION = "${params.REGION}"
            // we use 1114 user which will assume role in 1120 account in terraform
            AWS_CREDENTIALS_ID = 'aws-SA_gtp_deployment'
          }
          steps {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CREDENTIALS_ID]]) {
              sh """
                cd velero
                source create_s3_bucket.sh --role ${params.ENVIRONMENT} --workspace ${params.WORK} --region ${params.REGION}
              """
            }
          }
        }

        // Monitoring - Deploy Grafana operator
        stage('Monitoring - Deploy Grafana operator') {
          when { branch "master" }
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd grafana-operator
                source install.sh
                source test.sh
              """
          }
        }

        // Tracing - Jaeger Operator
        stage('Tracing - Jaeger Operator') {
          when { branch "master" }
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd jaeger-operator
                source install.sh
                source test.sh
              """
          }
        }
      // END Parallel 1
      }
    }    



    // BEGIN Parallel 2
    stage('Parallel 2') {
      when { branch "master" }
      parallel {

        // Install external-dns
        stage('Install External DNS') {
          when { branch "master" }
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
              // args "-u root"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
            AWS_DEFAULT_REGION = 'eu-central-1'
          }
          steps {
            sh """
              cd external-dns
              source install.sh
              # Comment test for now because it takes too much time (DNS propagation)
              # apk add --update bind-tools
              # source test.sh ${params.ENVIRONMENT}-${params.WORK}
            """
          }
        }

        // Install Velero
        stage('Install Velero') {
          when { branch "master" }
          agent {
            docker {
              image "${VELERO_IMAGE}"
              args "--entrypoint=''"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
            AWS_DEFAULT_REGION = "${params.REGION}"
          }
          steps {
            sh """
              cd velero
              source install.sh
              source test.sh
            """
          }
        }
      // END Parallel 2
      }
    }


    // Install Istio
    stage('Install Istio') {
    when { branch "master" }
      agent {
        docker {
          image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
          reuseNode true
          alwaysPull true
          registryUrl "https://${REPO_CI_PULL}"
          registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
        }
      }
      environment { 
        KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
      }
      steps {
          sh """
            cd istio
            source install.sh
          """
      }
    }

    // Monitoring - Deploy Prometheus operator
    stage('Monitoring - Deploy Prometheus operator') {
      when { branch "master" }
      agent {
        docker {
          image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
          reuseNode true
          alwaysPull true
          registryUrl "https://${REPO_CI_PULL}"
          registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
        }
      }
      environment { 
        KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
      }
      steps {
          sh """
            cd kube-prometheus
            source install.sh
          """
      }
    }


    // BEGIN Parallel 3
    stage('Parallel 3') {
      when { branch "master" }
      parallel {
        // Grafana - Create instance
        stage('Grafana - Create instance') {
          when { branch "master" }
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd grafana-instance
                source install.sh
                source test.sh
              """
          }
        }
        
        // Grafana - Import K8s dashboards
        stage('Grafana - Import K8S dashboards') {
          when { branch "master" }
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
              args "-u root"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd grafana-k8s-dashboards
                source install.sh
                source test.sh
              """
          }
        }

        // Grafana - Istio Federation & dashboards
        stage('Grafana - Istio Federation & dashboards') {
          when { branch "master" }
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd grafana-istio-dashboards
                source install.sh
                source test.sh
              """
          }
        }

        // Tracing - Jaeger Instance
        stage('Tracing - Jaeger Instance') {
          when { branch "master" }
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd jaeger-instance
                source install.sh
                source test.sh
              """
          }
        }

        // Create internal endpoint to access to GUIs
        stage('Terraform Internal Endpoint : Plan & Apply') {
          when { branch "master" }
          agent {
            docker {
              image "${REPO_CI_PULL}/cloud/continental/ehorizon/builds/terraform-aws:0.12.20"
              args "--entrypoint='' -u root"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
            AWS_CREDENTIALS_ID = 'aws-SA_gtp_deployment'
            AWS_DEFAULT_REGION = 'eu-central-1'
          }
          steps {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CREDENTIALS_ID]]) {
              sh """
                cd internal-endpoint
                source install.sh
                source test.sh
              """
            }
          }
        }

        // Deploy Custom Alerts
        stage('Deploy Custom Alerts') {
          when { branch "master" }
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              args "-u root"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd custom-alerts
                source install.sh
              """
          }
        }

        // Deploy MicroDemo application
        stage('Deploy Microservices Demo') {
          when { branch "master" }
          agent {
            docker {
              image "${REPO_CI_PULL}/${K8S_TOOLSET_IMAGE}"
              args "-u root"
              reuseNode true
              alwaysPull true
              registryUrl "https://${REPO_CI_PULL}"
              registryCredentialsId "$REPO_CI_PULL_CREDENTIALS_ID"
            }
          }
          environment { 
            KUBECONFIG = "${WORKSPACE}/kubeconfig.conf"
          }
          steps {
              sh """
                cd micro-demo
                source install.sh
              """
          }
        }

      // END Parallel 3
      }
    }

  }

  post {
		failure {
			slackSend color: 'danger', message: ":jenkins-devil: K8S post-install failed for ${params.ENVIRONMENT} environment ! (<${env.BUILD_URL}|#${env.BUILD_NUMBER}>) :jenkins-devil:"
		}
	}
}
