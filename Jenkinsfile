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

        stage('Esperar que inicien los pods') {
            agent { label 'minikube' }  
            steps {
                script {
                    def isRunning = false
                    
                    while (!isRunning) {
                        def podStatuses = sh(script: "minikube kubectl -- get pods -o jsonpath='{.items[*].status.phase}'", returnStdout: true).trim().split()

                        echo "Estados de los pods: ${podStatuses.join(', ')}"

                        isRunning = podStatuses.every { it == 'Running' }

                        if (!isRunning) {
                            echo "Esperando a que todos los pods estén en estado 'Running'..."
                            sleep 10 // Esperar 10 segundos antes de volver a verificar
                        }
                    }
                    //sh '''
                    //nohup minikube kubectl -- port-forward svc/appx-service 8080:8080 &
                    //'''
                }
            }
        }
        stage('Test Deployment') {
            agent { label 'minikube' }  
            steps {
                script {
                    sleep 15
                    sh '''
                    minikube kubectl -- get pods
                    curl http://localhost:8080/libros
                    '''
                }
            }
        }
    }
}
