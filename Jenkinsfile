pipeline {
    agent { label 'bijouterie' }

    options {
        timestamps()   // Horodate chaque ligne de la console
    }

    environment {
        DOCKERHUB_USER = 'ahmed372'

        // Jenkins récupère le nom d'utilisateur et le token
        // depuis les credentials configurés dans Jenkins.
        DOCKERHUB_CREDS = credentials('dockerhub-credentials')
    }

    stages {

        // Checkout : récupère le code du dépôt configuré dans Jenkins.
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // Build : construit une image Docker pour chaque microservice.
        // Chaque dossier constitue le contexte de construction.
        stage('Build') {
            steps {
                // Construction de l'image Backend.
                sh "docker build -t ${DOCKERHUB_USER}/bijouterie-backend:${BUILD_NUMBER} backend/"

                // Construction de l'image Frontend.
                sh "docker build -t ${DOCKERHUB_USER}/bijouterie-frontend:${BUILD_NUMBER} frontend/"

                // Construction de l'image MySQL.
                sh "docker build -t ${DOCKERHUB_USER}/bijouterie-mysql:${BUILD_NUMBER} database/"
            }
        }

        // Tag : ajoute le tag latest à chaque image construite.
        // BUILD_NUMBER permet de conserver une version propre à chaque build.
        stage('Tag') {
            steps {
                // Tag de l'image Backend.
                sh "docker tag ${DOCKERHUB_USER}/bijouterie-backend:${BUILD_NUMBER} ${DOCKERHUB_USER}/bijouterie-backend:latest"

                // Tag de l'image Frontend.
                sh "docker tag ${DOCKERHUB_USER}/bijouterie-frontend:${BUILD_NUMBER} ${DOCKERHUB_USER}/bijouterie-frontend:latest"

                // Tag de l'image MySQL.
                sh "docker tag ${DOCKERHUB_USER}/bijouterie-mysql:${BUILD_NUMBER} ${DOCKERHUB_USER}/bijouterie-mysql:latest"
            }
        }

        // Push : authentifie Jenkins auprès de Docker Hub,
        // puis publie les deux tags de chaque image.
        stage('Push') {
            steps {
                // Connexion sécurisée à Docker Hub avec le token Jenkins.
                sh 'echo "$DOCKERHUB_CREDS_PSW" | docker login -u "$DOCKERHUB_CREDS_USR" --password-stdin'

                // Publication de l'image Backend.
                sh "docker push ${DOCKERHUB_USER}/bijouterie-backend:${BUILD_NUMBER}"
                sh "docker push ${DOCKERHUB_USER}/bijouterie-backend:latest"

                // Publication de l'image Frontend.
                sh "docker push ${DOCKERHUB_USER}/bijouterie-frontend:${BUILD_NUMBER}"
                sh "docker push ${DOCKERHUB_USER}/bijouterie-frontend:latest"

                // Publication de l'image MySQL.
                sh "docker push ${DOCKERHUB_USER}/bijouterie-mysql:${BUILD_NUMBER}"
                sh "docker push ${DOCKERHUB_USER}/bijouterie-mysql:latest"
            }
        }
    }

    // Post : s'exécute même si un stage échoue.
    // Déconnexion de Docker Hub après le pipeline.
    post {
        always {
            sh 'docker logout || true'
        }
    }
}