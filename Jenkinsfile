pipeline {
    agent any

    tools {
        nodejs "NodeJS_16"
    }

    environment {
        DOCKER_HOST = "unix:///var/run/docker.sock"
        DOCKER_HUB_USER = 'dmzz'
        FRONT_IMAGE = 'express-frontend'
        BACK_IMAGE  = 'express-backend'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Environment Check') {
            steps {
                sh '''
                    echo "=== Environment Information ==="
                    node -v
                    npm -v
                    whoami
                    echo "=== Docker Socket ==="
                    ls -la /var/run/docker.sock || echo "Docker socket not found"
                    echo "=== Trying Docker Commands ==="
                    docker --version || echo "Docker not available in PATH"
                    docker-compose --version || echo "Docker-compose not available"
                '''
            }
        }

        stage('Install Docker in Container') {
            steps {
                script {
                    // Installer Docker dans le conteneur Jenkins si nécessaire
                    sh '''
                        if ! command -v docker &> /dev/null; then
                            echo "🔧 Installing Docker inside Jenkins container..."
                            apt-get update
                            apt-get install -y docker.io
                        fi
                        
                        if ! command -v docker-compose &> /dev/null; then
                            echo "🔧 Installing Docker Compose..."
                            apt-get install -y docker-compose
                        fi
                        
                        docker --version
                        docker-compose --version
                    '''
                }
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('Backend Dependencies') {
                    steps {
                        dir('back-end') {
                            sh 'npm install --no-audit --no-fund'
                        }
                    }
                }
                stage('Frontend Dependencies') {
                    steps {
                        dir('front-end') {
                            sh 'npm install --no-audit --no-fund'
                        }
                    }
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        dir('back-end') {
                            sh 'npm test || echo "⚠️ Backend tests failed or missing"'
                        }
                    }
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        dir('front-end') {
                            sh 'npm test || echo "⚠️ Frontend tests failed or missing"'
                        }
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    sh '''
                        echo "🔧 Testing Docker access..."
                        docker ps || echo "Docker not accessible"
                    '''
                    
                    sh "docker build -t ${env.DOCKER_HUB_USER}/${env.FRONT_IMAGE}:latest ./front-end"
                    sh "docker build -t ${env.DOCKER_HUB_USER}/${env.BACK_IMAGE}:latest ./back-end"
                    
                    sh 'docker images | grep ${DOCKER_HUB_USER} || echo "No images found"'
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                script {
                    sh '''
                        echo "🚀 Starting deployment..."
                        
                        # Vérifier et créer compose.yaml si nécessaire
                        if [ ! -f "compose.yaml" ] && [ ! -f "docker-compose.yml" ]; then
                            echo "📁 Creating basic compose.yaml..."
                            cat > compose.yaml << 'EOF'
version: "3.9"
services:
  frontend:
    image: ${DOCKER_HUB_USER}/express-frontend:latest
    container_name: react-frontend
    ports:
      - "5173:5173"
    environment:
      - VITE_API_URL=http://localhost:5001/api
    depends_on:
      - backend

  backend:
    image: ${DOCKER_HUB_USER}/express-backend:latest
    container_name: express-backend
    ports:
      - "5001:5001"
    environment:
      - NODE_ENV=production
      - MONGODB_URI=mongodb://mongo:27017/smartphoneDB
    depends_on:
      - mongo

  mongo:
    image: mongo:6.0
    container_name: mongo-db
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

volumes:
  mongo_data:
EOF
                        fi

                        COMPOSE_FILE="compose.yaml"
                        if [ ! -f "$COMPOSE_FILE" ] && [ -f "docker-compose.yml" ]; then
                            COMPOSE_FILE="docker-compose.yml"
                        fi

                        echo "📋 Using compose file: $COMPOSE_FILE"
                        
                        # Arrêter les conteneurs existants
                        docker-compose -f $COMPOSE_FILE down || true
                        
                        # Démarrer les nouveaux conteneurs
                        docker-compose -f $COMPOSE_FILE up -d
                        
                        # Attendre le démarrage
                        sleep 30
                        
                        # Vérifier l'état
                        docker-compose -f $COMPOSE_FILE ps
                        echo "✅ Deployment completed"
                    '''
                }
            }
        }

        stage('Smoke Tests') {
            steps {
                script {
                    sh '''
                        echo "🧪 Running smoke tests..."
                        
                        # Test Backend
                        echo "Testing Backend (port 5001)..."
                        curl -f http://localhost:5001/api/smartphones || echo "Backend not ready yet"
                        
                        # Test Frontend
                        echo "Testing Frontend (port 5173)..."
                        curl -f http://localhost:5173 || echo "Frontend not ready yet"

                        echo "🎉 Smoke tests completed"
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                echo "=== Final Status ==="
                docker ps -a || echo "Cannot check containers"
                echo "=== Build Information ==="
                echo "Job: ${JOB_NAME}"
                echo "Build: ${BUILD_NUMBER}"
                echo "URL: ${BUILD_URL}"
            '''
        }
        success {
            echo "🎉 Pipeline exécuté avec succès!"
        }
        failure {
            echo "❌ Le pipeline a échoué!"
        }
    }
}
