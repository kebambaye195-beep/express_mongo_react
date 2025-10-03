pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io/dmzz' // Ex: docker.io/yourusername
        PROJECT_NAME = 'smartphone-app'
        FRONTEND_IMAGE = "${DOCKER_REGISTRY}/${PROJECT_NAME}-frontend"
        BACKEND_IMAGE = "${DOCKER_REGISTRY}/${PROJECT_NAME}-backend"
        MONGO_IMAGE = 'mongo:6.0'
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    currentBuild.displayName = "BUILD-${env.BUILD_NUMBER}"
                    currentBuild.description = "Branch: ${env.GIT_BRANCH}"
                }
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('Frontend Lint') {
                    when {
                        changeset "front-end/**"
                    }
                    steps {
                        dir('front-end') {
                            sh 'npm install'
                            sh 'npm run lint || true' // Continue even if lint fails
                        }
                    }
                }
                stage('Backend Lint') {
                    when {
                        changeset "back-end/**"
                    }
                    steps {
                        dir('back-end') {
                            sh 'npm install'
                            sh 'npm run lint || true'
                        }
                    }
                }
            }
        }
        
        stage('Tests') {
            parallel {
                stage('Frontend Tests') {
                    when {
                        changeset "front-end/**"
                    }
                    steps {
                        dir('front-end') {
                            sh 'npm test -- --watchAll=false --coverage --passWithNoTests'
                        }
                    }
                    post {
                        always {
                            publishHTML(target: [
                                allowMissing: true,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'front-end/coverage/lcov-report',
                                reportFiles: 'index.html',
                                reportName: 'Frontend Coverage'
                            ])
                        }
                    }
                }
                stage('Backend Tests') {
                    when {
                        changeset "back-end/**"
                    }
                    steps {
                        dir('back-end') {
                            sh 'npm test -- --coverage --passWithNoTests'
                        }
                    }
                    post {
                        always {
                            publishHTML(target: [
                                allowMissing: true,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'back-end/coverage/lcov-report',
                                reportFiles: 'index.html',
                                reportName: 'Backend Coverage'
                            ])
                        }
                    }
                }
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    // Build Frontend
                    docker.build("${FRONTEND_IMAGE}:${env.BUILD_NUMBER}", "-f front-end/Dockerfile ./front-end")
                    
                    // Build Backend
                    docker.build("${BACKEND_IMAGE}:${env.BUILD_NUMBER}", "-f back-end/Dockerfile ./back-end")
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    // Scan Frontend image (requires trivy installed on Jenkins agent)
                    sh "trivy image --exit-code 0 --severity HIGH,CRITICAL ${FRONTEND_IMAGE}:${env.BUILD_NUMBER} || true"
                    
                    // Scan Backend image
                    sh "trivy image --exit-code 0 --severity HIGH,CRITICAL ${BACKEND_IMAGE}:${env.BUILD_NUMBER} || true"
                }
            }
        }
        
        stage('Push to Registry') {
            when {
                branch 'main' // or 'master', 'develop' selon votre workflow
            }
            steps {
                script {
                    // Login to Docker Registry (configure credentials in Jenkins)
                    docker.withRegistry('https://' + env.DOCKER_REGISTRY, 'docker-credentials') {
                        docker.image("${FRONTEND_IMAGE}:${env.BUILD_NUMBER}").push()
                        docker.image("${BACKEND_IMAGE}:${env.BUILD_NUMBER}").push()
                        
                        // Also push latest tag for main branch
                        if (env.BRANCH_NAME == 'main') {
                            docker.image("${FRONTEND_IMAGE}:${env.BUILD_NUMBER}").push('latest')
                            docker.image("${BACKEND_IMAGE}:${env.BUILD_NUMBER}").push('latest')
                        }
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Déploiement sur l'environnement de staging
                    sh """
                    docker-compose -f docker-compose.staging.yml down
                    docker-compose -f docker-compose.staging.yml pull
                    docker-compose -f docker-compose.staging.yml up -d
                    """
                }
            }
        }
        
        stage('Integration Tests') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Attendre que les services soient démarrés
                    sh 'sleep 30'
                    
                    // Exécuter des tests d'intégration
                    dir('back-end') {
                        sh 'npm run test:integration || true'
                    }
                    
                    // Tests E2E pour le frontend
                    dir('front-end') {
                        sh 'npm run test:e2e || true'
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Nettoyage des images Docker locales
            sh 'docker system prune -f || true'
            
            // Archivage des artefacts
            archiveArtifacts artifacts: '**/coverage/**/*, **/build/**/*', allowEmptyArchive: true
        }
        success {
            emailext (
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Le build ${env.BUILD_URL} est réussi",
                to: "${env.CHANGE_AUTHOR_EMAIL}",
                attachLog: false
            )
        }
        failure {
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Le build ${env.BUILD_URL} a échoué",
                to: "${env.CHANGE_AUTHOR_EMAIL}",
                attachLog: true
            )
        }
        unstable {
            emailext (
                subject: "UNSTABLE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Le build ${env.BUILD_URL} est instable",
                to: "${env.CHANGE_AUTHOR_EMAIL}",
                attachLog: true
            )
        }
    }
}
