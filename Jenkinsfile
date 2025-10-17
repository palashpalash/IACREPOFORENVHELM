pipeline {
  agent any

  parameters {
    choice(name: 'ENV', choices: ['dev', 'stage', 'uat'], description: 'Target environment')
    string(name: 'REPO_NAME', defaultValue: 'https://github.com/palashpalash/IACREPOFORENVHELM.git', description: 'GitHub repo name OR full URL')
    string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to deploy')
    string(name: 'IMAGE_REPO', defaultValue: 'dockerhubusername/reponame', description: 'Docker Hub repo (e.g. palash123567/my-springboot-app)')
    string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Image tag to deploy')
    string(name: 'RELEASE_NAME', defaultValue: 'myapp', description: 'Helm release name')
    string(name: 'REGION', defaultValue: 'ap-south-1', description: 'AWS region')
  }

  environment {
    // Leave empty to use EC2 instance profile on the agent.
    // Set to your Jenkins Credentials ID (AccessKey/Secret) to use explicit creds.
    AWS_CREDS = ''   // e.g. 'aws-credentials-id'
  }

  options {
    timestamps()
    ansiColor('xterm')
  }

  stages {
    stage('Checkout') {
      steps {
        script {
          def url = params.REPO_NAME
          if (!url.startsWith('http')) {
            url = "https://github.com/${params.GITHUB_USER}/${params.REPO_NAME}.git"
          }
          echo "Cloning: ${url} @ ${params.BRANCH}"
          checkout([$class: 'GitSCM',
            branches: [[name: params.BRANCH]],
            userRemoteConfigs: [[url: url]]
          ])
        }
        sh 'test -d helm || { echo "helm/ folder missing"; exit 1; }'
      }
    }

    stage('Select Cluster') {
      steps {
        script {
          def envToCluster = [
            dev:   'eks_dev',
            stage: 'eks_stage',
            uat:   'eks_uat'
          ]
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
          def runAws = { cmd ->
            if (env.AWS_CREDS?.trim()) {
              withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: env.AWS_CREDS]]) {
                sh cmd
              }
            } else {
              sh cmd
            }
          }

          runAws """
            aws sts get-caller-identity
            aws eks update-kubeconfig --name ${env.CLUSTER} --region ${params.REGION}
            kubectl config current-context
          """
          sh "kubectl get ns"
        }
      }
    }

    stage('Helm Lint & Dry Run') {
      steps {
        sh """
          helm lint helm
          helm template ${params.RELEASE_NAME} helm \
            -f helm/values-${params.ENV}.yaml \
            --set image.repository=${params.IMAGE_REPO} \
            --set image.tag=${params.IMAGE_TAG} \
            --namespace ${env.NAMESPACE} | head -n 100
        """
      }
    }

    stage('Deploy') {
      steps {
        sh """
          helm upgrade --install ${params.RELEASE_NAME} helm \
            -f helm/values-${params.ENV}.yaml \
            --set image.repository=${params.IMAGE_REPO} \
            --set image.tag=${params.IMAGE_TAG} \
            --namespace ${env.NAMESPACE} \
            --history-max 10 \
            --wait --timeout 10m
        """
      }
    }

    stage('Post-Deploy Check') {
      steps {
        sh """
          kubectl -n ${env.NAMESPACE} get deploy,po,svc -l app.kubernetes.io/instance=${params.RELEASE_NAME}
          echo "LB Hostname:"
          kubectl -n ${env.NAMESPACE} get svc -l app.kubernetes.io/instance=${params.RELEASE_NAME} \
            -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}{"\\n"}' || true
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
