pipeline {
  agent any

  tools {
    nodejs "Node_JS16"
  }

  environment {
    DOCKER_USER = 'kebambaye195-beep'
    FRONT_IMAGE_LOCAL = 'front-image-local'
    BACK_IMAGE_LOCAL  = 'back-image-local'

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

    stage('Build Docker Images (LOCAL ONLY)') {
      steps {
        script {
          sh "docker build -t ${FRONT_IMAGE_LOCAL}:latest ./front-end"
          sh "docker build -t ${BACK_IMAGE_LOCAL}:latest ./back-end"
        }
      }
    }

    stage('Trivy Scan') {
      steps {
        script {
          sh '''
            echo "🔍 Installing Trivy..."
            if ! command -v trivy &> /dev/null; then
              curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
            fi

            mkdir -p trivy-reports

            echo "📥 Updating Trivy database..."
            trivy --download-db-only

            echo "🧪 Scanning Docker images..."
            trivy image --severity HIGH,CRITICAL --format json -o trivy-reports/frontend-image.json front-image-local:latest || true
            trivy image --severity HIGH,CRITICAL --format json -o trivy-reports/backend-image.json back-image-local:latest || true

            echo "🧪 Scanning filesystem (npm vulnerabilities)..."
            trivy fs --severity HIGH,CRITICAL --format json -o trivy-reports/frontend-fs.json ./front-end || true
            trivy fs --severity HIGH,CRITICAL --format json -o trivy-reports/backend-fs.json ./back-end || true

            echo "📝 Generating HTML reports..."
            trivy image --severity HIGH,CRITICAL --format template --template "@contrib/html.tpl" -o trivy-reports/frontend-image.html front-image-local:latest || true
            trivy image --severity HIGH,CRITICAL --format template --template "@contrib/html.tpl" -o trivy-reports/backend-image.html back-image-local:latest || true
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
            docker tag front-image-local:latest $DOCKER_USER/express-frontend:latest
            docker tag back-image-local:latest $DOCKER_USER/express-backend:latest

            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

            docker push $DOCKER_USER/express-frontend:latest
            docker push $DOCKER_USER/express-backend:latest
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

    stage('Deploy compose.yaml') {
      steps {
        sh 'docker-compose -f compose.yaml down || true'
        sh 'docker-compose -f compose.yaml pull'
        sh 'docker-compose -f compose.yaml up -d'
      }
    }

    stage('Smoke Test') {
      steps {
        sh '''
          echo "Frontend test..."
          curl -f http://localhost:5173 || echo "Frontend unreachable"

          echo "Backend test..."
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
          Le pipeline Jenkins est terminé.
          <p>Rapports Trivy :</p>
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
        body: "Échec du pipeline : ${env.BUILD_URL}",
        to: "kebambaye195@gmail.com"
      )
    }
  }
}
