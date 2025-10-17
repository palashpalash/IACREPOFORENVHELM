pipeline {
  agent any

  parameters {
    choice(name: 'ENV', choices: ['dev', 'stage', 'uat'], description: 'Target environment')
    string(name: 'IMAGE_REPO', defaultValue: 'dockerhubusername/reponame', description: 'Docker Hub repo (e.g. palash123567/my-springboot-app)')
    string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Image tag to deploy')
    string(name: 'RELEASE_NAME', defaultValue: 'myapp', description: 'Helm release name')
    string(name: 'REGION', defaultValue: 'ap-south-1', description: 'AWS region')
  }

  options { timestamps() }

  environment {
    // Leave empty to use the agent's IAM role (instance profile).
    // If you need explicit AWS creds, put the Jenkins Credentials ID here.
    AWS_CREDS = ''
  }

  stages {
    stage('Checkout (from SCM)') {
      steps {
        // Use the repo Jenkins already checked out for this job
        checkout scm
        // Verify chart exists
        powershell 'if (!(Test-Path helm)) { throw "helm/ folder missing" }'
        powershell 'Get-ChildItem helm'
      }
    }

    stage('Select Cluster') {
      steps {
        script {
          def envToCluster = [ dev: 'eks_dev', stage: 'eks_stage', uat: 'eks_uat' ]
          env.CLUSTER   = envToCluster[params.ENV]
          env.NAMESPACE = 'default'
          if (!env.CLUSTER) error "Unknown ENV=${params.ENV}"
          echo "Target: cluster=${env.CLUSTER}, namespace=${env.NAMESPACE}"
        }
      }
    }

    stage('Auth & Kube Context') {
      steps {
        script {
          def runAws = { psCmd ->
            if (env.AWS_CREDS?.trim()) {
              withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: env.AWS_CREDS]]) {
                powershell(psCmd)
              }
            } else {
              powershell(psCmd)
            }
          }
          runAws """
            aws sts get-caller-identity
            aws eks update-kubeconfig --name ${env.CLUSTER} --region ${params.REGION}
            kubectl config current-context
            kubectl get ns
          """
        }
      }
    }

    stage('Helm Lint & Dry Run') {
      steps {
        powershell """
          helm lint helm
          helm template ${params.RELEASE_NAME} helm `
            -f helm/values-${params.ENV}.yaml `
            --set image.repository=${params.IMAGE_REPO} `
            --set image.tag=${params.IMAGE_TAG} `
            --namespace ${env.NAMESPACE} | Select-Object -First 80
        """
      }
    }

    stage('Deploy') {
      steps {
        powershell """
          helm upgrade --install ${params.RELEASE_NAME} helm `
            -f helm/values-${params.ENV}.yaml `
            --set image.repository=${params.IMAGE_REPO} `
            --set image.tag=${params.IMAGE_TAG} `
            --namespace ${env.NAMESPACE} `
            --history-max 10 `
            --wait --timeout 10m
        """
      }
    }

    stage('Post-Deploy Check') {
      steps {
        powershell """
          kubectl -n ${env.NAMESPACE} get deploy,po,svc -l app.kubernetes.io/instance=${params.RELEASE_NAME}
          Write-Host 'LB Hostname:'
          kubectl -n ${env.NAMESPACE} get svc -l app.kubernetes.io/instance=${params.RELEASE_NAME} `
            -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}{\"`n\"}' | Out-String
        """
      }
    }
  }

  post {
    success {
      echo "✅ Deployed ${params.RELEASE_NAME} to ${params.ENV} (${env.CLUSTER}) with ${params.IMAGE_REPO}:${params.IMAGE_TAG}"
    }
    failure {
      echo "❌ Deployment failed. Check logs above."
    }
  }
}
