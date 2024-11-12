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
                    cd /home/$USER/dds-deploy/k8s-appx/
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
                    sh '''
                    while [[ $(minikube kubectl -- get pods --no-headers | awk '{print $2}' | grep -v '1/1' | wc -l) -ne 0 ]]; do
                        echo "Esperando a que los pods estén 'Running' y 'Ready'..."
                        sleep 10
                    done
        
                    # Verificar que no hay pods en estado 'Pending', 'Failed' o que no están 'Running'
                    while [[ $(minikube kubectl -- get pods --field-selector=status.phase!=Running --no-headers | wc -l) -ne 0 ]]; do
                        echo "Esperando a que los pods estén en estado 'Running'..."
                        sleep 10
                    done
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
                    curl -X POST  $(minikube service appx-service --url)/libros \
                     -H "Content-Type: application/json" \
                     -d '{"nombre": "El bar del infierno", "autor": "Alejandro Dolina", "precio": 2000}'
                    curl $(minikube service appx-service --url)/libros
                    '''
                }
            }
        }
    }
}
