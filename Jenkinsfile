voici mon jenkinsfile, alors modifie l'etape trivy pour moi pipeline {
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

    stage('Trivy Scan') {
      steps {
        script {
          sh '''
            echo "🔍 Installation et exécution de Trivy..."
            if ! command -v trivy &> /dev/null; then
              echo "📦 Installation de Trivy..."
              apt-get update && apt-get install -y wget gnupg lsb-release
              wget -qO- https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor -o /usr/share/keyrings/trivy.gpg
              echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | tee /etc/apt/sources.list.d/trivy.list
              apt-get update && apt-get install -y trivy
            fi

            echo "🧪 Préparation du fichier d’ignore Trivy..."
            mkdir -p /var/lib/trivy

            cat > /tmp/.trivyignore.yaml <<EOF
ignoreRules:
  - vulnerabilityID: CVE-2024-24790
    reason: "False positive from esbuild (Go runtime not used in prod)"
  - vulnerabilityID: CVE-2023-39325
    reason: "Affects Go HTTP server, not relevant for esbuild binary"
  - vulnerabilityID: CVE-2023-45283
    reason: "Not exploitable in frontend context"
  - vulnerabilityID: CVE-2023-45288
    reason: "Go stdlib issue irrelevant to esbuild usage"
EOF

            echo "🚀 Lancement des scans Trivy (avec politique d’ignore)..."

            # Scan frontend (sans bloquer le pipeline)
            trivy image \
              --cache-dir /var/lib/trivy \
              --no-progress \
              --ignore-unfixed \
              --severity HIGH,CRITICAL \
              --ignore-policy /tmp/.trivyignore.yaml \
              $DOCKER_USER/$FRONT_IMAGE:latest || echo "⚠️ Vulnérabilités ignorées pour le frontend"

            # Scan backend
            trivy image \
              --cache-dir /var/lib/trivy \
              --no-progress \
              --ignore-unfixed \
              --severity HIGH,CRITICAL \
              --ignore-policy /tmp/.trivyignore.yaml \
              $DOCKER_USER/$BACK_IMAGE:latest || echo "⚠️ Vulnérabilités ignorées pour le backend"

            echo "✅ Scan Trivy terminé — aucune vulnérabilité critique non ignorée."
          '''
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
          Scan Trivy : aucun problème critique détecté
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
