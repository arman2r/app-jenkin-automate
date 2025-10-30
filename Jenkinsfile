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
                    // Notifica inicio del build
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
                    def gitCommit = bat(returnStdout: true, script: '@git rev-parse --short HEAD').trim()
                    def gitAuthor = bat(returnStdout: true, script: '@git log -1 --pretty=format:%%an').trim()
                    def gitMessage = bat(returnStdout: true, script: '@git log -1 --pretty=format:%%s').trim()
                    
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
                bat 'npm ci'
            }
        }
        
        stage('Lint') {
            steps {
                echo '🔍 Ejecutando linter...'
                bat 'npm run lint'
            }
        }
        
        stage('Build') {
            steps {
                echo '🔨 Construyendo aplicación Next.js...'
                bat 'npm run build'
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
                    bat '''
                        @echo off
                        FOR /F "tokens=5" %%P IN ('netstat -aon ^| findstr :3000 ^| findstr LISTENING') DO TaskKill /PID %%P /F 2>nul
                        echo Instancia anterior detenida
                    '''
                }
            }
        }
        
        stage('Deploy') {
            steps {
                echo '🚀 Desplegando aplicación...'
                script {
                    bat 'start /B npm run start'
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
                        def response = bat(returnStatus: true, script: 'curl -f http://localhost:3000')
                        if (response == 0) {
                            echo '✅ Aplicación respondiendo correctamente'
                        }
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