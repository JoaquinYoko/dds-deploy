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
                    cd /home/yoko/dds-deploy/k8s-appx/
                    git pull
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
                        def podStatuses = sh(script: "minikube kubectl -- get pods --no-headers -o custom-columns=NAME:.metadata.name,STATUS:.status.phase", returnStdout: true).trim()

                        echo "Estados de los pods:\n${podStatuses}"

                        // Verificar si todos los pods están "Running"
                        def allRunning = podStatuses.split('\n').every { line ->
                            line.split()[1] == 'Running'
                        }

                        if (allRunning) {
                            isRunning = true
                        } else {
                            echo "Esperando a que todos los pods estén en estado 'Running'..."
                            sleep 10
                        }
                    }
                }
            }
        }
        stage('Test Deployment') {
            agent { label 'minikube' }  
            steps {
                script {
                    sh '''
                    minikube kubectl -- get pods
                    while [[ $(minikube kubectl -- get pods --no-headers | awk '{print $2}' | grep -v '1/1' | wc -l) -ne 0 ]]; do
                        echo "Esperando a que los pods estén 'Running' y 'Ready'..."
                        sleep 10
                    done
        
                    # Verificar que no hay pods en estado 'Pending', 'Failed' o que no están 'Running'
                    while [[ $(minikube kubectl -- get pods --field-selector=status.phase!=Running --no-headers | wc -l) -ne 0 ]]; do
                        echo "Esperando a que los pods estén en estado 'Running'..."
                        sleep 10
                    done
                    curl $(minikube service appx-service --url)/libros
                    '''
                }
            }
        }
    }
}
