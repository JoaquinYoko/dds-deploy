pipeline {
    agent none 
    
    tools {
        maven 'maven'  // Nombre de la instalación configurada en Global Tool Configuration
    }
    
    stages { 
        stage('SCM Checkout') {
            agent { label 'master' }
            steps{
           git branch: 'main', url: 'https://github.com/JoaquinYoko/dds-deploy.git'
            }
        }
        stage('Compilar') {
            agent { label 'master' }
            steps {
                sh 'mvn clean install'
            }
        }
        // run sonarqube test
        stage('Run Sonarqube') {
            agent { label 'master' }
            environment {
                scannerHome = tool 'sonar-tool';
            }
            steps {
              withSonarQubeEnv(credentialsId: 'jnkns', installationName: 'sonar-server') {
                sh "${scannerHome}/bin/sonar-scanner"
              }
            }
        }
        
         stage('Setup Minikube') {
            agent { label 'minikube' }  
            steps {
                script {
                    sh '''
                    minikube start --driver=docker
                    '''
                }
            }
        }
        
        stage('Deploy App en Minikube') {
            agent { label 'minikube' }  
            steps {
                script {
                    sh '''
                    cd /home/yoko/k8s-appx/
                    minikube kubectl -- apply -f db-deployment.yaml
                    minikube kubectl -- apply -f app-deployment.yaml
                    '''
                }
            }
        }

        stage('Test Deployment') {
            agent { label 'minikube' }  
            steps {
                script {
                    sh '''
                    minikube kubectl -- get pods
                    '''
                }
            }
        }
    }
}
