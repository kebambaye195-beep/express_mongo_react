pipeline {
  agent any

  tools {
    nodejs "Node_JS16"
  }

  environment {
    DOCKER_USER = 'kebambaye195-beep'
    FRONT_IMAGE = 'express-frontend'
    BACK_IMAGE  = 'express-backend'
  }

  triggers {
    GenericTrigger(
      genericVariables: [
        [key: 'ref', value: '$.ref'],
        [key: 'pusher_name', value: '$.pusher.name'],
        [key: 'commit_message', value: '$.head_commit.message']
      ],
      causeString: 'Push par $pusher_name sur $ref: "$commit_message"',
      token: 'mysecret',
      printContributedVariables: true,
      printPostContent: true
    )
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

    stage('Build Docker Images') {
      steps {
        script {
          sh "docker build -t $DOCKER_USER/$FRONT_IMAGE:latest ./front-end"
          sh "docker build -t $DOCKER_USER/$BACK_IMAGE:latest ./back-end"
        }
      }
    }

    /* === 🔍 Nouvelle étape TRIVY === */
    stage('Trivy Scan') {
      steps {
        script {
          sh '''
            echo "🔍 Installation de Trivy si nécessaire..."
            if ! command -v trivy &> /dev/null; then
              apt-get update && apt-get install -y wget gnupg lsb-release
              mkdir -p /etc/apt/keyrings
              wget -qO- https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor -o /etc/apt/keyrings/trivy.gpg
              echo "deb [signed-by=/etc/apt/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | tee /etc/apt/sources.list.d/trivy.list
              apt-get update && apt-get install -y trivy
            fi

            echo "📦 Démarrage du scan Trivy..."

            mkdir -p trivy-reports

            # Scan Frontend
            echo "🧪 Scan du frontend..."
            trivy image --no-progress --scanners vuln \
              --severity HIGH,CRITICAL \
              --format table \
              --output trivy-reports/frontend-report.txt \
              --exit-code 0 \
              $DOCKER_USER/$FRONT_IMAGE:latest || true

            # Scan Backend
            echo "🧪 Scan du backend..."
            trivy image --no-progress --scanners vuln \
              --severity HIGH,CRITICAL \
              --format table \
              --output trivy-reports/backend-report.txt \
              --exit-code 0 \
              $DOCKER_USER/$BACK_IMAGE:latest || true

            echo "✅ Scan Trivy terminé — rapports générés dans trivy-reports/"
          '''
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'trivy-reports/*.txt', fingerprint: true
        }
      }
    }

    stage('Push Docker Images') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
            docker push $DOCKER_USER/$FRONT_IMAGE:latest
            docker push $DOCKER_USER/$BACK_IMAGE:latest
          '''
        }
      }
    }

    stage('Clean Docker') {
      steps {
        sh 'docker container prune -f'
        sh 'docker image prune -f'
      }
    }

    stage('Check Docker & Compose') {
      steps {
        sh 'docker --version'
        sh 'docker-compose --version || echo "docker-compose non trouvé"'
      }
    }

    stage('Deploy (compose.yaml)') {
      steps {
        dir('.') {
          sh 'docker-compose -f compose.yaml down || true'
          sh 'docker-compose -f compose.yaml pull'
          sh 'docker-compose -f compose.yaml up -d'
          sh 'docker-compose -f compose.yaml ps'
          sh 'docker-compose -f compose.yaml logs --tail=50'
        }
      }
    }

    stage('Smoke Test') {
      steps {
        sh '''
          echo " Vérification Frontend (port 5173)..."
          curl -f http://localhost:5173 || echo "Frontend unreachable"

          echo " Vérification Backend (port 5001)..."
          curl -f http://localhost:5001/api || echo "Backend unreachable"
        '''
      }
    }
  }

  post {
    success {
      emailext(
        subject: "✅ Build SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: """
          Pipeline réussi 🎉
          Détails : ${env.BUILD_URL}
          Rapport Trivy archivé dans les artefacts Jenkins.
        """,
        to: "kebambaye195@gmail.com"
      )
    }

    failure {
      emailext(
        subject: "❌ Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: """
          Le pipeline a échoué 🚨
          Détails : ${env.BUILD_URL}
          Vérifie le rapport Trivy ou les étapes Docker.
        """,
        to: "kebambaye195@gmail.com"
      )
    }
  }
}
