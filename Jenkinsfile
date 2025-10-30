pipeline {
    agent any
    
    tools {
        nodejs 'NodeJS 22.x'
    }
    
    environment {
        NODE_ENV = 'production'
        APP_NAME = 'my-app-nextjs'
        APP_PORT = '3000'
    }
    
    triggers {
        pollSCM('H/5 * * * *')
        githubPush()
    }
    
    stages {
        stage('Inicio') {
            steps {
                script {
                    currentBuild.description = "Build #${env.BUILD_NUMBER} - Iniciado"
                }
                echo '🚀 Iniciando pipeline de deployment...'
            }
        }
        
        stage('Checkout') {
            steps {
                echo '📦 Clonando repositorio...'
                checkout scm
                
                script {
                    def gitCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    def gitAuthor = sh(returnStdout: true, script: 'git log -1 --pretty=format:%an').trim()
                    def gitMessage = sh(returnStdout: true, script: 'git log -1 --pretty=format:%s').trim()
                    
                    echo "👤 Autor: ${gitAuthor}"
                    echo "📝 Commit: ${gitCommit}"
                    echo "💬 Mensaje: ${gitMessage}"
                    
                    currentBuild.description = "${gitMessage} (${gitCommit})"
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo '📥 Instalando dependencias...'
                sh 'npm ci'
            }
        }
        
        stage('Lint') {
            steps {
                echo '🔍 Ejecutando linter...'
                sh 'npm run lint'
            }
        }
        
        stage('Build') {
            steps {
                echo '🔨 Construyendo aplicación Next.js...'
                sh 'npm run build'
            }
        }
        
        stage('Test') {
            steps {
                echo '🧪 Ejecutando tests...'
                echo 'No hay tests configurados aún'
            }
        }
        
        stage('Stop Previous Instance') {
            steps {
                echo '🛑 Deteniendo instancia anterior...'
                script {
                    sh '''
                        # Buscar y matar procesos en el puerto 3000
                        PID=$(lsof -ti:3000) || true
                        if [ ! -z "$PID" ]; then
                            kill -9 $PID
                            echo "Instancia anterior detenida (PID: $PID)"
                        else
                            echo "No hay instancia anterior corriendo"
                        fi
                    '''
                }
            }
        }
        
        stage('Deploy') {
            steps {
                echo '🚀 Desplegando aplicación...'
                script {
                    sh '''
                        # Iniciar la aplicación en segundo plano
                        nohup npm run start > /var/jenkins_home/workspace/vision-fe/app.log 2>&1 &
                        echo $! > /var/jenkins_home/workspace/vision-fe/app.pid
                        echo "Aplicación iniciada con PID: $(cat /var/jenkins_home/workspace/vision-fe/app.pid)"
                    '''
                    sleep(time: 10, unit: 'SECONDS')
                    echo '✅ Aplicación desplegada en http://localhost:3000'
                }
            }
        }
        
        stage('Health Check') {
            steps {
                echo '❤️ Verificando salud de la aplicación...'
                script {
                    retry(3) {
                        sleep(time: 2, unit: 'SECONDS')
                        sh 'curl -f http://localhost:3000 || exit 0'
                        echo '✅ Aplicación respondiendo correctamente'
                    }
                }
            }
        }
    }
    
    post {
        success {
            script {
                def duration = currentBuild.durationString.replace(' and counting', '')
                echo '========================================='
                echo '✅ DEPLOYMENT EXITOSO'
                echo '========================================='
                echo "⏱️  Duración: ${duration}"
                echo "🔢 Build: #${env.BUILD_NUMBER}"
                echo "🌐 URL: http://localhost:3000"
                echo "📅 ${new Date().format('dd/MM/yyyy HH:mm:ss')}"
                echo '========================================='
                
                currentBuild.result = 'SUCCESS'
            }
        }
        
        failure {
            script {
                echo '========================================='
                echo '❌ DEPLOYMENT FALLIDO'
                echo '========================================='
                echo "🔢 Build: #${env.BUILD_NUMBER}"
                echo "📅 ${new Date().format('dd/MM/yyyy HH:mm:ss')}"
                echo '========================================='
                
                currentBuild.result = 'FAILURE'
            }
        }
        
        unstable {
            echo '⚠️ Build inestable'
        }
        
        always {
            echo '🏁 Pipeline finalizado'
            echo "Estado final: ${currentBuild.result}"
        }
    }
}