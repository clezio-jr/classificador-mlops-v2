pipeline {
  agent { label 'built-in' }
  parameters { string(name: 'ROLLBACK_TAG', defaultValue: '', description: 'Tag já publicada (ex.: v1.0.0)') }
  options { disableConcurrentBuilds(abortPrevious: true) }
  environment {
    IMAGE = 'ghcr.io/clezio-jr/classificador-mlops-v2'
    TAG   = "${params.ROLLBACK_TAG ?: env.TAG_NAME ?: ''}"
  }

  stages {
    stage('Testes') {
      when { expression { !params.ROLLBACK_TAG } }
      agent { docker { image 'python:3.11-slim'; reuseNode true } }
      steps {
        sh 'pip install -r requirements-dev.txt'
        sh 'python -m pytest -m "not e2e" --cov=src --cov-report=term-missing'
      }
    }

    stage('Build') {
      when { changeRequest() }
      steps { sh 'docker build -t intent-clf:ci .' }
    }

    stage('Smoke') {
      when { allOf { expression { !params.ROLLBACK_TAG }; anyOf { branch 'main'; buildingTag() } } }
      environment {
        WANDB_API_KEY = credentials('WANDB_API_KEY')
        WANDB_PROJECT = credentials('WANDB_PROJECT')
        WANDB_MODELS  = credentials('WANDB_MODELS')
        MONGO_URL     = credentials('MONGO_URL')
        MONGO_DB      = 'ci_tests'
      }
      stages {
        stage('Subir API') { steps { sh 'docker compose up -d --build' } }
        stage('e2e') {
          agent { docker { image 'python:3.11'; args '--network container:intent-clf'; reuseNode true } }
          steps {
            sh 'curl -fs -o /dev/null --retry 30 --retry-delay 10 --retry-all-errors http://localhost:8000/'
            sh 'pip install -r requirements-dev.txt'
            sh 'E2E_BASE_URL=http://localhost:8000 python -m pytest -m e2e -ra'
          }
        }
      }
      post {
        failure { sh 'docker compose logs --tail=200' }
        cleanup { sh 'docker compose down -v' }
      }
    }

    stage('Release') {
      when { tag 'v*' }
      environment { GHCR = credentials('GHCR') }
      steps {
        sh 'echo "$GHCR_PSW" | docker login ghcr.io -u "$GHCR_USR" --password-stdin'
        sh 'docker build -t "$IMAGE:$TAG" .'
        sh 'docker push "$IMAGE:$TAG"'
      }
    }

    stage('CD') {
      when { anyOf { tag 'v*'; expression { params.ROLLBACK_TAG as boolean } } }
      input { message 'Fazer deploy em produção?' }
      stages {
        stage('Deploy') {
          agent { docker { image 'mcr.microsoft.com/azure-cli'; reuseNode true } }
          environment {
            AZURE_CLIENT_ID     = credentials('AZURE_CLIENT_ID')
            AZURE_CLIENT_SECRET = credentials('AZURE_CLIENT_SECRET')
            AZURE_TENANT_ID     = credentials('AZURE_TENANT_ID')
          }
          steps {
            sh 'az login --service-principal -u "$AZURE_CLIENT_ID" -p="$AZURE_CLIENT_SECRET" --tenant "$AZURE_TENANT_ID" -o none'
            sh 'az containerapp update -n intent-clf -g mlops-2026-2 --image "$IMAGE:$TAG" --set-env-vars APP_VERSION="$TAG"'
          }
        }
        stage('Verify') {
          steps {
            retry(30) {
              sleep 10
              sh 'curl -fsS "$APP_URL/" | grep -q "\\"$TAG\\""'
              sh '''curl -fsS -X POST "$APP_URL/predictions" -H "Content-Type: application/json" -d '{"nome":"cd","text":"não entendi nada"}' '''
            }
          }
        }
      }
    }
  }
}
