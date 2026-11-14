pipeline {
    // Definition du label de l'agent Jenkins (agent Windows lance sur le PC)
    agent { label 'tools' }

    // Definition des options du pipeline
    options {
        // Le checkout est fait explicitement dans le stage Checkout
        skipDefaultCheckout()
    }

    // Definition des variables d'environnement (noms des images sur Docker Hub)
    environment {
        FRONTEND_IMAGE = 'ahmed372/bijouterie-frontend'
        BACKEND_IMAGE  = 'ahmed372/bijouterie-backend'
        MYSQL_IMAGE    = 'ahmed372/bijouterie-mysql'
    }

    stages {

        // Recuperation du code source depuis GitHub
        stage('Checkout') {
            steps {
                echo 'Recuperation du code source depuis GitHub'
                checkout scm
            }
        }

        // Verification de Docker et Git sur l'agent
        stage('Verification') {
            steps {
                bat 'docker version'
                bat 'git --version'
            }
        }

        // Construction des trois images Docker (le contexte est le dossier du service)
        stage('Build') {
            steps {
                echo 'Construction des trois images Docker'
                bat 'docker build -t bijouterie-frontend:build .\\frontend'
                bat 'docker build -t bijouterie-backend:build .\\backend'
                bat 'docker build -t bijouterie-mysql:build .\\database'
            }
        }

        // Double tag : le numero de build (utilise dans les manifests) et latest
        // latest sert seulement sur Docker Hub, il est interdit dans k8s/
        stage('Tag') {
            steps {
                echo "Attribution des tags ${BUILD_NUMBER} et latest"
                bat 'docker tag bijouterie-frontend:build %FRONTEND_IMAGE%:%BUILD_NUMBER%'
                bat 'docker tag bijouterie-frontend:build %FRONTEND_IMAGE%:latest'
                bat 'docker tag bijouterie-backend:build %BACKEND_IMAGE%:%BUILD_NUMBER%'
                bat 'docker tag bijouterie-backend:build %BACKEND_IMAGE%:latest'
                bat 'docker tag bijouterie-mysql:build %MYSQL_IMAGE%:%BUILD_NUMBER%'
                bat 'docker tag bijouterie-mysql:build %MYSQL_IMAGE%:latest'
            }
        }

        // Connexion a Docker Hub avec le credential Jenkins, puis publication des images
        stage('Push') {
            steps {
                echo 'Publication des images sur Docker Hub'
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    // Guillemets simples : PowerShell lit le secret, Groovy ne l'interpole pas.
                    // --password-stdin evite d'afficher le mot de passe dans la commande.
                    powershell '$env:DH_PASS | docker login -u $env:DH_USER --password-stdin'
                }
                bat 'docker push %FRONTEND_IMAGE%:%BUILD_NUMBER%'
                bat 'docker push %FRONTEND_IMAGE%:latest'
                bat 'docker push %BACKEND_IMAGE%:%BUILD_NUMBER%'
                bat 'docker push %BACKEND_IMAGE%:latest'
                bat 'docker push %MYSQL_IMAGE%:%BUILD_NUMBER%'
                bat 'docker push %MYSQL_IMAGE%:latest'
            }
        }

        // Mise a jour automatique des manifests Kubernetes (GitOps : Argo CD deploie ensuite)
        stage('Update manifests') {
            steps {
                script {
                    echo "Mise a jour des manifests avec le tag ${BUILD_NUMBER}"
                    // Le Deployment MySQL utilise l'image officielle mysql:8.4 (bonus init.sql),
                    // il n'est donc pas dans cette liste.
                    def cibles = [
                        'k8s/06-backend-deployment.yaml'  : env.BACKEND_IMAGE,
                        'k8s/09-frontend-deployment.yaml' : env.FRONTEND_IMAGE
                    ]
                    cibles.each { fichier, image ->
                        def contenu = readFile(file: fichier, encoding: 'UTF-8')
                        // [^\\s#]+ remplace seulement le tag et conserve le commentaire de fin de ligne
                        def motif = "(?m)(image:\\s*${java.util.regex.Pattern.quote(image)}:)[^\\s#]+"
                        def remplacement = '$1' + env.BUILD_NUMBER
                        def nouveauContenu = contenu.replaceAll(motif, remplacement)
                        if (!nouveauContenu.contains("image: ${image}:${env.BUILD_NUMBER}")) {
                            error("Le remplacement de l'image a echoue dans ${fichier}")
                        }
                        writeFile(file: fichier, text: nouveauContenu, encoding: 'UTF-8')
                        echo "Manifest mis a jour : ${fichier}"
                    }
                }
                // Configuration de l'auteur du commit
                bat 'git config user.name "Jenkins"'
                bat 'git config user.email "jenkins@users.noreply.github.com"'
                // Ajout des fichiers modifies
                bat 'git add k8s/06-backend-deployment.yaml k8s/09-frontend-deployment.yaml'
                // Verification des modifications, commit puis push
                script {
                    def changements = bat(script: '@git diff --cached --quiet', returnStatus: true)
                    if (changements == 0) {
                        echo 'Aucune modification a committer.'
                    } else {
                        bat 'git commit -m "ci: mise a jour des images - build %BUILD_NUMBER% [skip ci]"'
                        withCredentials([gitUsernamePassword(credentialsId: 'github-credentials', gitToolName: 'Default')]) {
                            bat 'git push origin HEAD:main'
                        }
                    }
                }
            }
        }
    }

    // Deconnexion de Docker Hub, meme si un stage a echoue
    post {
        always {
            bat(script: 'docker logout', returnStatus: true)
        }
    }
}