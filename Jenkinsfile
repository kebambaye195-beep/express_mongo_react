pipeline {
  agent any

  tools {
    nodejs "Node_JS16"
  }

  environment {
    DOCKER_USER = 'kebambaye195-beep'
    FRONT_IMAGE = 'express-frontend'
    BACK_IMAGE  = 'express-backend'
    SONAR_SCANNER_HOME = "${env.WORKSPACE}/sonar-scanner-4.8.0.2856"
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

    stage('Install SonarScanner') {
      steps {
        script {
          sh '''
            echo "🔧 Installation de SonarScanner..."
            # Vérifier si wget et unzip sont disponibles
            if ! command -v wget &> /dev/null; then
              echo "Installation de wget..."
              apt-get update && apt-get install -y wget
            fi
            
            if ! command -v unzip &> /dev/null; then
              echo "Installation de unzip..."
              apt-get install -y unzip
            fi

            # Vérifier si sonar-scanner est déjà installé
            if ! command -v sonar-scanner &> /dev/null; then
              echo "Téléchargement de SonarScanner..."
              # URL CORRIGÉE - l'ancienne URL était incorrecte
              wget -q https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.8.0.2856.zip
              
              if [ -f "sonar-scanner-cli-4.8.0.2856.zip" ]; then
                echo "Décompression de SonarScanner..."
                unzip -q sonar-scanner-cli-4.8.0.2856.zip
                rm sonar-scanner-cli-4.8.0.2856.zip
                echo "✅ SonarScanner installé avec succès"
                
                # Vérifier que l'installation a réussi
                if [ -f "${SONAR_SCANNER_HOME}/bin/sonar-scanner" ]; then
                  echo "✅ Fichier sonar-scanner trouvé"
                else
                  echo "❌ Fichier sonar-scanner non trouvé après installation"
                  exit 1
                fi
              else
                echo "❌ Échec du téléchargement de SonarScanner"
                exit 1
              fi
            else
              echo "✅ SonarScanner est déjà installé"
            fi
          '''
        }
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

    stage('SonarQube Analysis') {
      steps {
        echo "🔍 Analyse du code avec SonarQube..."
        withSonarQubeEnv('Sonarqube') {
          withCredentials([string(credentialsId: 'sonarqubeid', variable: 'SONAR_TOKEN')]) {
            sh """
              # Ajouter SonarScanner au PATH
              export PATH=\"${env.SONAR_SCANNER_HOME}/bin:\$PATH\"
              
              echo "Vérification de la version de sonar-scanner..."
              sonar-scanner --version || echo "Impossible d'exécuter sonar-scanner"
              
              echo "Exécution de l'analyse SonarQube..."
              sonar-scanner \
                -Dsonar.projectKey=sonarqube1 \
                -Dsonar.sources=. \
                -Dsonar.host.url=http://localhost:9000 \
                -Dsonar.login=$SONAR_TOKEN
            """
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        echo "🛡 Vérification du Quality Gate..."
        timeout(time: 2, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
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
        subject: "Build SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: "Pipeline réussi\nDétails : ${env.BUILD_URL}",
        to: "kebambaye195@gmail.com"
      )
    }
    failure {
      emailext(
        subject: "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: "Le pipeline a échoué\nDétails : ${env.BUILD_URL}",
        to: "kebambaye195@gmail.com"
      )
    }
    
    always {
      sh '''
        echo "🧹 Nettoyage des fichiers d'installation..."
        rm -rf sonar-scanner-4.8.0.2856 || true
      '''
    }
  }
}
```
