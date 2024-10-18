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
                    sh '''
                    nohup minikube kubectl -- port-forward svc/appx-service 8080:8080 > /dev/null 2>&1 &
                    echo $! > port_forward_pid.txt  # Guardar el PID para detenerlo más tarde si es necesario
                    '''
                    def isPortForwarded = false
                    while (!isPortForwarded) {
                        def result = sh(script: "lsof -i :8080", returnStatus: true)
                        if (result == 0) {
                            echo "Port-forwarding exitoso en el puerto 8080"
                            isPortForwarded = true
                        } else {
                            echo "Esperando a que el port-forwarding se establezca..."
                            sleep 5 // Esperar 5 segundos antes de volver a verificar
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
                    curl http://localhost:8080/libros
                    '''
                }
            }
        }
    }
}
