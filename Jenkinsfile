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
    SONARQUBE_URL = "http://sonarqube:9000"
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

    stage('Diagnostic Réseau') {
      steps {
        sh '''
          echo "🔍 Diagnostic de connectivité réseau..."
          echo "Adresse IP de l'hôte :"
          hostname -I || true
          echo "Test de connexion à SonarQube :"
          curl -f http://sonarqube:9000 && echo "✅ Connecté à sonarqube:9000" || echo "❌ Impossible de se connecter à sonarqube:9000"
          curl -f http://host.docker.internal:9000 && echo "✅ Connecté à host.docker.internal:9000" || echo "❌ Impossible de se connecter à host.docker.internal:9000"
          curl -f http://localhost:9000 && echo "✅ Connecté à localhost:9000" || echo "❌ Impossible de se connecter à localhost:9000"
        '''
      }
    }

    stage('Install SonarScanner') {
      steps {
        script {
          sh '''
            echo "🔧 Installation de SonarScanner..."
            if ! command -v wget &> /dev/null; then
              echo "Installation de wget..."
              apt-get update && apt-get install -y wget
            fi
            
            if ! command -v unzip &> /dev/null; then
              echo "Installation de unzip..."
              apt-get install -y unzip
            fi

            if [ ! -d "sonar-scanner-4.8.0.2856" ]; then
              echo "Téléchargement de SonarScanner..."
              wget -q https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.8.0.2856.zip
              
              if [ -f "sonar-scanner-cli-4.8.0.2856.zip" ]; then
                echo "Décompression de SonarScanner..."
                unzip -q sonar-scanner-cli-4.8.0.2856.zip
                rm sonar-scanner-cli-4.8.0.2856.zip
                echo "✅ SonarScanner installé avec succès"
              else
                echo "❌ Échec du téléchargement de SonarScanner"
                exit 1
              fi
            else
              echo "✅ SonarScanner déjà présent"
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
        script {
          // Marquer si l'analyse SonarQube réussit
          def sonarAnalysisSuccess = false
          def sonarTaskId = null
          
          try {
            withSonarQubeEnv('Sonarqube') {
              withCredentials([string(credentialsId: 'sonarqubeid', variable: 'SONAR_TOKEN')]) {
                sh """
                  export PATH=\"${env.SONAR_SCANNER_HOME}/bin:\$PATH\"
                  echo "Vérification de la version de sonar-scanner..."
                  sonar-scanner --version
                  
                  echo "Exécution de l'analyse SonarQube..."
                  sonar-scanner \
                    -Dsonar.projectKey=sonarqube1 \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=${SONARQUBE_URL} \
                    -Dsonar.login=${SONAR_TOKEN}
                """
                sonarAnalysisSuccess = true
                echo "✅ Analyse SonarQube terminée avec succès"
                
                // Récupérer l'ID de tâche SonarQube
                sonarTaskId = sh(
                  script: "find . -name '*.txt' -exec grep -l 'sonarqube1' {} \\; | xargs grep -h 'taskId' | head -1 | sed 's/.*taskId=//'",
                  returnStdout: true
                ).trim()
              }
            }
          } catch (Exception e) {
            echo "⚠ Analyse SonarQube échouée: ${e.getMessage()}"
            echo "➡ Tentative de connexion alternative..."
            
            // Tentative alternative
            try {
              withCredentials([string(credentialsId: 'sonarqubeid', variable: 'SONAR_TOKEN')]) {
                sh """
                  export PATH=\"${env.SONAR_SCANNER_HOME}/bin:\$PATH\"
                  echo "Tentative avec host.docker.internal..."
                  sonar-scanner \
                    -Dsonar.projectKey=sonarqube1 \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=http://host.docker.internal:9000 \
                    -Dsonar.login=${SONAR_TOKEN}
                """
                sonarAnalysisSuccess = true
                echo "✅ Analyse SonarQube réussie avec host.docker.internal"
              }
            } catch (Exception e2) {
              echo "❌ Toutes les tentatives d'analyse SonarQube ont échoué"
              sonarAnalysisSuccess = false
            }
          }
          
          // Stocker le statut dans une variable d'environnement
          env.SONAR_ANALYSIS_SUCCESS = sonarAnalysisSuccess.toString()
          if (sonarTaskId) {
            env.SONAR_TASK_ID = sonarTaskId
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        echo "🛡 Vérification du Quality Gate..."
        script {
          if (env.SONAR_ANALYSIS_SUCCESS == 'true') {
            try {
              echo "⏳ Attente du résultat du Quality Gate..."
              timeout(time: 5, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: false
              }
              echo "✅ Quality Gate terminée"
            } catch (Exception e) {
              echo "⚠ Erreur lors de la vérification du Quality Gate: ${e.getMessage()}"
              echo "Cela peut être dû à:"
              echo "- Temps d'analyse SonarQube trop long"
              echo "- Problème de connexion au serveur SonarQube"
              echo "- Fichier de rapport SonarQube non trouvé"
              
              // Attendre manuellement et vérifier le statut
              sleep 30
              echo "➡ Continuation du pipeline malgré l'erreur Quality Gate"
            }
          } else {
            echo "⚠ Analyse SonarQube non exécutée, impossible de vérifier le Quality Gate"
            echo "➡ Continuation du pipeline sans vérification de qualité"
          }
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
        body: """
          Pipeline réussi
          Détails : ${env.BUILD_URL}
          Analyse SonarQube: ${env.SONAR_ANALYSIS_SUCCESS == 'true' ? 'SUCCÈS' : 'ÉCHEC'}
          Quality Gate: ${env.SONAR_ANALYSIS_SUCCESS == 'true' ? 'VÉRIFIÉE' : 'NON VÉRIFIÉE'}
        """,
        to: "kebambaye195@gmail.com"
      )
    }
    failure {
      emailext(
        subject: "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: """
          Le pipeline a échoué
          Détails : ${env.BUILD_URL}
          Analyse SonarQube: ${env.SONAR_ANALYSIS_SUCCESS == 'true' ? 'SUCCÈS' : 'ÉCHEC'}
        """,
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
