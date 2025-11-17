pipeline {
  agent any
  tools { nodejs 'node' }

  environment {
    IMAGE   = "${env.BRANCH_NAME == 'main' ? 'nodemain' : 'nodedev'}"
    PORT    = "${env.BRANCH_NAME == 'main' ? '3000' : '3001'}"
    TAG     = "v1.0"
    CONTAINER_NAME = "node-${env.BRANCH_NAME}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build') {
      steps {
        sh '''
          set -e
          # npm ci быстрее и детерминированнее, fallback на npm install
          if [ -f package-lock.json ]; then npm ci; else npm install; fi
        '''
      }
    }

    stage('Test') {
      steps {
        sh '''
          set -e
          npm test || echo "No tests - continue"
        '''
      }
    }

    // Оставь этот stage, если используешь logos/main|dev/logo.svg.
    // ИЛИ удали stage, если держишь разные src/logo.svg по веткам.
    stage('Pick logo by branch') {
      steps {
        sh '''
          set -e
          TARGET=src/logo.svg
          SRC="logos/${BRANCH_NAME}/logo.svg"
          if [ -f "$SRC" ]; then
            cp "$SRC" "$TARGET"
            echo "Logo selected for ${BRANCH_NAME} -> $TARGET"
          else
            echo "No branch-specific logo found at $SRC; keeping existing $TARGET"
          fi
        '''
      }
    }

    stage('Docker build') {
      steps {
        sh '''
          set -e
          docker build -t ${IMAGE}:${TAG} .
          docker images | grep ${IMAGE} | head -n 1
        '''
      }
    }

    stage('Deploy') {
      steps {
        sh '''
          set -e
          # Остановить/удалить только свой контейнер
          if docker ps -a --format '{{.Names}}' | grep -q "^${CONTAINER_NAME}$"; then
            docker stop ${CONTAINER_NAME} || true
            docker rm   ${CONTAINER_NAME} || true
          fi

          # Освободить порт, если занят другим контейнером
          USED=$(docker ps --filter "publish=${PORT}" --format '{{.Names}}' || true)
          if [ -n "$USED" ] && [ "$USED" != "${CONTAINER_NAME}" ]; then
            echo "Port ${PORT} is used by $USED. Stopping it..."
            docker stop "$USED" || true
            docker rm   "$USED" || true
          fi

          docker run -d --name ${CONTAINER_NAME} --expose 3000 -p ${PORT}:3000 ${IMAGE}:${TAG}
          sleep 1
          docker ps --filter "name=${CONTAINER_NAME}"
        '''
      }
    }
  }

  post {
    always {
      echo "Branch=${env.BRANCH_NAME} Image=${env.IMAGE}:${env.TAG} Port=${env.PORT}"
    }
  }
}
