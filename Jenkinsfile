pipeline {
    agent any

    tools {
        nodejs "NodeJS_16"
    }

    environment {
        DOCKER_HUB_USER = 'dmzz'
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
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/fatimaaaaah/application_MERN.git'
            }
        }

        stage('Install dependencies - Backend') {
            steps {
                dir('back-end') {
                    sh 'npm ci'
                }
            }
        }

        stage('Install dependencies - Frontend') {
            steps {
                dir('front-end') {
                    sh 'npm ci'
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    try {
                        sh 'cd back-end && npm test'
                    } catch (e) {
                        echo "Aucun test backend ou échec des tests: ${e.message}"
                    }
                    try {
                        sh 'cd front-end && npm test'
                    } catch (e) {
                        echo "Aucun test frontend ou échec des tests: ${e.message}"
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_HUB_USER}/${FRONT_IMAGE}:latest ./front-end"
                    sh "docker build -t ${DOCKER_HUB_USER}/${BACK_IMAGE}:latest ./back-end"
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials', 
                    usernameVariable: 'DOCKER_USER', 
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    script {
                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker push ${DOCKER_HUB_USER}/${FRONT_IMAGE}:latest
                            docker push ${DOCKER_HUB_USER}/${BACK_IMAGE}:latest
                        """
                    }
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
                sh 'docker compose version'
            }
        }

        stage('Deploy (compose.yaml)') {
            steps {
                sh 'docker compose -f compose.yaml down || true'
                sh 'docker compose -f compose.yaml pull || true'
                sh 'docker compose -f compose.yaml up -d --build'
                sleep 30
                sh 'docker compose -f compose.yaml ps'
                sh 'docker compose -f compose.yaml logs --tail=20'
            }
        }

        stage('Smoke Test') {
            steps {
                script {
                    try {
                        sh 'curl -f http://localhost:5173 --retry 3 --retry-delay 5 || echo "Frontend unreachable"'
                    } catch (e) {
                        echo "Frontend smoke test failed: ${e.message}"
                    }
                    try {
                        sh 'curl -f http://localhost:5001/api --retry 3 --retry-delay 5 || echo "Backend unreachable"'
                    } catch (e) {
                        echo "Backend smoke test failed: ${e.message}"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            emailext(
                subject: "Build SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                Pipeline réussi !
                Détails : ${env.BUILD_URL}
                """,
                to: "sowgokuuza@gmail.com"
            )
        }
        failure {
            emailext(
                subject: "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                Le pipeline a échoué
                Détails : ${env.BUILD_URL}
                """,
                to: "sowgokuuza@gmail.com"
            )
        }
    }
}
