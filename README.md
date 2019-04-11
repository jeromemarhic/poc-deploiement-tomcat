# Jenkins

docker pull stephenreed/jenkins-java8-maven-git
docker run -d -p 15003:8080 stephenreed/jenkins-java8-maven-git

Récupérer le mot de passe par défaut dans le fichier /jenkins/secrets/initialAdminPassword

Ouvrir un navigateur sur http://127.0.0.1:15003
Indiquer le mot de passe récupérer en amont
Choississez Sélectionner les plugins à installer et ajouter
    Conditional BuildStep
    Multijob
    Copy Artifact
    Publish Over SSH
    SSH
Créer le 1er utilisateur Administrateur
    Nom : jenkins
    Mot de passe  : jenkins
Menu Jenkins / Administrater Jenkins
    Mettre à jour la version de Jenkins
Menu Jenkins / Identifiants / System / Identifiants globaux
    Ajouter des identifiants
        Type : Nom d'utilisateur et mot de passe
        Portée : Global
        Nom d'utilisateur : the-manager
        Mot de passe : needs-a-new-password-here
        ID : tomcat-manager
        Description : Tomcat manager pour déployer les wars
Menu Jenkins / Administrater Jenkins / Configuration globale des outils
    Ajouter JDK
        Nom : JDK 8
        Install automatically : Yes
        Version : JAVA SE DevelopmentKit 8u*
    Ajouter Maven
        Nom : Maven 3
        Install automatically : Yes
        Version : 3.*.*
Menu Jenkins / Administrer Jenkins / Gestion des plugins
    Installer "Deploy to container"
Menu Jenkins / Nouvel Item
    Nom : Build and Deploy
    Pipeline

pipeline {
    tools {
        maven "Maven 3"
        jdk "JDK 8"
    }
    agent any 
    stages {
        stage('Build') {
            steps {
                echo 'Build wars'
                echo 'Build war API'
                dir("ApiBuildDir") {
                    git changelog: false, poll: false, url: 'https://github.com/drims-poke/poc-deploiement-api', branch: 'master'
                    sh 'mvn clean install'
                }
                echo 'Archive API'
                archiveArtifacts 'ApiBuildDir/target/*.war'
            }
        }
        stage('Turn maintenance on') {
            steps {
                echo 'Turn maintenance on' 
                echo 'TODO CHANGE APACHE TO MAINTENANCE ON' 
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploy Database'
                dir("DataBaseDeployDir") {
                    git changelog: false, poll: false, url: 'https://github.com/drims-poke/poc-deploiement-liquibase', branch: 'master'
                    sh 'mvn liquibase:update -Dip=172.17.0.1 -Dport=15001 -Ddatabase=api_db -Dusername=root -Dpassword=database-password'
                }

                echo 'Deploy Tomcat Settings'
                echo 'TODO STOP TOMCAT'
                echo 'TODO COPY CONTEXT.XML'
                echo 'TODO START TOMCAT'

                echo 'Deploy wars API'
                echo 'Deploy API'
                sh 'curl -s --upload-file ApiBuildDir/target/api*.war "http://the-manager:needs-a-new-password-here@172.17.0.1:15002/manager/text/deploy?path=/api&update=true"'
            }
        }
        stage('Turn maintenance off') {
            steps {
                echo 'Turn maintenance off' 
                echo 'TODO CHANGE APACHE TO MAINTENANCE ON' 
            }
        }
    }
}

docker commit 1397ef8425c0 stephenreed/jenkins-java8-maven-git:deploy

Par la suite, pour démarrer Jenkins : docker run -d -p 15003:8080 stephenreed/jenkins-java8-maven-git:deploy

# MySQL

docker pull mysql:8
docker run -d -p 15001:3306 -e MYSQL_ROOT_PASSWORD=database-password mysql:8

# Tomcat

docker pull picoded/tomcat
docker run -d -p 15002:8080 picoded/tomcat

# Installation SSH

Sur le serveur tomcat :
apt-get update
apt-get install openssh-server
ssh-keygen (génère la paire de clés)
Dans ~/.ssh, renommer 'id_rsa.pub' en 'authorized_keys'
Modifier le fichier /etc/ssh/sshd.config :
    - PermitRootLogin Yes
