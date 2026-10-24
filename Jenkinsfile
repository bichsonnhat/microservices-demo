// Online Boutique (fork microservices-demo) — chỉ build service có thay đổi trong src/<service>/
//   branch dev  -> Harbor boutique/<svc>:dev-<sha>  -> boutique-gitops envs/dev/<svc>/values.yaml -> ns boutique-dev
//   branch main -> Harbor boutique/<svc>:prod-<sha> -> boutique-gitops envs/prod/<svc>/values.yaml-> ns boutique-prod
pipeline {
  agent any
  parameters {
    booleanParam(name: 'BUILD_ALL', defaultValue: false, description: 'Build lại TẤT CẢ service (bỏ qua phát hiện thay đổi)')
  }
  options {
    disableConcurrentBuilds()
    timeout(time: 90, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }
  environment {
    REGISTRY     = '172.31.9.183:8085'
    PROJECT      = 'boutique'
    GITOPS_REPO  = 'github.com/bichsonnhat/boutique-gitops.git'
    ALL_SERVICES = 'adservice cartservice checkoutservice currencyservice emailservice frontend loadgenerator paymentservice productcatalogservice recommendationservice shippingservice'
  }
  stages {
    stage('Init') {
      steps {
        script {
          env.DEPLOY_ENV = ['main': 'prod', 'dev': 'dev'][env.BRANCH_NAME] ?: ''
          env.SHORT_SHA  = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.TAG        = "${env.DEPLOY_ENV ?: 'ci'}-${env.SHORT_SHA}"
          env.PREV       = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT ?: ''
          env.FORCE_ALL  = params.BUILD_ALL ? 'true' : 'false'
          // Lần build đầu / commit cũ không còn / BUILD_ALL -> build tất cả; ngược lại chỉ src/<svc> thay đổi
          env.SERVICES = sh(returnStdout: true, script: '''#!/bin/bash
            if [ "$FORCE_ALL" = "true" ] || [ -z "$PREV" ] || ! git cat-file -e "$PREV^{commit}" 2>/dev/null; then
              echo "$ALL_SERVICES"
            else
              git diff --name-only "$PREV" "$GIT_COMMIT" \
                | awk -F/ '$1=="src" && NF>2 {print $2}' | sort -u \
                | grep -xF -f <(printf '%s\\n' $ALL_SERVICES) | tr '\\n' ' ' || true
            fi
          ''').trim()
          echo "Branch=${env.BRANCH_NAME} | env=${env.DEPLOY_ENV ?: '(không deploy)'} | tag=${env.TAG}"
          echo "So với commit: ${env.PREV ?: '(chưa có build thành công)'}"
          echo "Service sẽ build: ${env.SERVICES ?: '(không có)'}"
        }
      }
    }

    stage('Build & Push') {
      when { expression { return env.DEPLOY_ENV != '' && env.SERVICES != '' } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor', usernameVariable: 'HARBOR_USR', passwordVariable: 'HARBOR_PSW')]) {
          sh '''#!/bin/bash
            set -euo pipefail
            echo "$HARBOR_PSW" | docker login "$REGISTRY" -u "$HARBOR_USR" --password-stdin
            for S in $SERVICES; do
              CTX="src/$S"
              [ "$S" = "cartservice" ] && CTX="src/cartservice/src"     # Dockerfile của cartservice nằm ở đây
              echo "=============== build $S ($CTX) ==============="
              docker build -t "$REGISTRY/$PROJECT/$S:$TAG" "$CTX"
              docker push "$REGISTRY/$PROJECT/$S:$TAG"
            done
          '''
        }
      }
    }

    stage('Update GitOps') {
      when { expression { return env.DEPLOY_ENV != '' && env.SERVICES != '' } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'GH_USR', passwordVariable: 'GH_PSW')]) {
          sh '''#!/bin/bash
            set -euo pipefail
            rm -rf gitops
            git clone "https://${GH_USR}:${GH_PSW}@${GITOPS_REPO}" gitops
            cd gitops
            git config user.name  "Jenkins"
            git config user.email "jenkins@boutique.local"
            for S in $SERVICES; do
              VFILE="envs/${DEPLOY_ENV}/${S}/values.yaml"          # mỗi service 1 thư mục
              [ -f "$VFILE" ] || { echo "Bỏ qua $S: chưa có $VFILE"; continue; }
              yq -i ".image.tag = \\"${TAG}\\"" "$VFILE"
              git add "$VFILE"
            done
            git commit -m "boutique(${DEPLOY_ENV}): ${SERVICES} -> ${TAG}" || { echo "Không có thay đổi"; exit 0; }
            for i in 1 2 3; do git pull --rebase && git push && exit 0; sleep 5; done
            exit 1
          '''
        }
      }
    }
  }
  post {
    always {
      sh '''
        docker logout "$REGISTRY" || true
        docker image prune -af --filter "until=72h" || true
        docker builder prune -af --filter "until=72h" || true
      '''
    }
  }
}
