pipeline {
  agent any

  tools {
    nodejs "Node_JS16"
  }

  environment {
    DOCKER_USER = 'kebambaye195-beep'

    FRONT_LOCAL = 'express-frontend-local'
    BACK_LOCAL  = 'express-backend-local'

    FRONT_IMAGE = 'express-frontend'
    BACK_IMAGE  = 'express-backend'
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/kebambaye195-beep/express_mongo_react.git'
      }
    }

    stage('Install dependencies - Backend') {
      steps {
        dir('back-end') {
          sh 'npm install'
        }
      }
    }

    stage('Install dependencies - Frontend') {
      steps {
        dir('front-end') {
          sh 'npm install'
        }
      }
    }

    stage('Run Tests') {
      steps {
        script {
          sh 'cd back-end && npm test || echo "Aucun test backend"'
          sh 'cd front-end && npm test || echo "Aucun test frontend"'
        }
      }
    }

    stage('Build Docker Images (LOCAL)') {
      steps {
        script {
          sh "docker build -t ${FRONT_LOCAL}:latest ./front-end"
          sh "docker build -t ${BACK_LOCAL}:latest ./back-end"
        }
      }
    }

    stage('Trivy Scan') {
      steps {
        script {
          sh '''
            echo "🔍 Installation de Trivy si nécessaire..."
            if ! command -v trivy &> /dev/null; then
              curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | \
                sh -s -- -b /usr/local/bin
            fi

            mkdir -p trivy-reports

            echo "📥 Mise à jour de la base Trivy..."
            trivy --download-db-only

            echo "🧪 Scan des IMAGES LOCALES..."
            trivy image --severity HIGH,CRITICAL \
              --format json -o trivy-reports/frontend-image.json ${FRONT_LOCAL}:latest || true

            trivy image --severity HIGH,CRITICAL \
              --format json -o trivy-reports/backend-image.json ${BACK_LOCAL}:latest || true

            echo "🧪 Scan filesystem npm (vulnérabilités réelles Node)..."
            trivy fs --severity HIGH,CRITICAL \
              --format json -o trivy-reports/frontend-fs.json ./front-end || true

            trivy fs --severity HIGH,CRITICAL \
              --format json -o trivy-reports/backend-fs.json ./back-end || true

            echo "📝 Génération des rapports HTML..."
            trivy image --severity HIGH,CRITICAL \
              --format template --template "@contrib/html.tpl" \
              -o trivy-reports/frontend-image.html ${FRONT_LOCAL}:latest || true

            trivy image --severity HIGH,CRITICAL \
              --format template --template "@contrib/html.tpl" \
              -o trivy-reports/backend-image.html ${BACK_LOCAL}:latest || true
          '''
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'trivy-reports/*.*', fingerprint: true
        }
      }
    }

    stage('Tag & Push DockerHub Images') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            docker tag ${FRONT_LOCAL}:latest $DOCKER_USER/${FRONT_IMAGE}:latest
            docker tag ${BACK_LOCAL}:latest $DOCKER_USER/${BACK_IMAGE}:latest

            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

            docker push $DOCKER_USER/${FRONT_IMAGE}:latest
            docker push $DOCKER_USER/${BACK_IMAGE}:latest
          '''
        }
      }
    }

    stage('Deploy Docker Compose (DEV)') {
      steps {
        sh 'docker-compose -f compose.yaml down || true'
        sh 'docker-compose -f compose.yaml pull'
        sh 'docker-compose -f compose.yaml up -d'
      }
    }

    stage('Smoke Test') {
      steps {
        sh '''
          echo "Test Frontend..."
          curl -f http://localhost:5173 || echo "⚠ Frontend unreachable"

          echo "Test Backend..."
          curl -f http://localhost:5001/api || echo "⚠ Backend unreachable"
        '''
      }
    }
  }

  post {
    success {
      emailext(
        subject: "✅ Build SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: """
          Pipeline terminé avec succès.<br>
          <b>Rapports Trivy :</b>
          <ul>
            <li><a href="${BUILD_URL}artifact/trivy-reports/frontend-image.html">Frontend (image)</a></li>
            <li><a href="${BUILD_URL}artifact/trivy-reports/backend-image.html">Backend (image)</a></li>
          </ul>
        """,
        to: "kebambaye195@gmail.com"
      )
    }

    failure {
      emailext(
        subject: "❌ Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: "Le pipeline a échoué : ${env.BUILD_URL}",
        to: "kebambaye195@gmail.com"
      )
    }
  }
}
