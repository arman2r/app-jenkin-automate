pipeline {
    agent any

    environment {
        APP_NAME = 'my-app-nextjs'
        APP_PORT = '3000'
        N8N_WEBHOOK = 'http://192.168.1.10:5678/webhook/jenkins-events' 
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Inicio') {
            steps {
                script {
                    // Configurar NodeJS manualmente si tools no funciona
                    def nodeHome = tool name: 'NodeJS 22.x', type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
                    env.PATH = "${nodeHome}/bin:${env.PATH}"

                    currentBuild.description = "Build #${env.BUILD_NUMBER} - Iniciado"
                }

                echo '🚀 Iniciando pipeline de deployment...'

                // Notificar inicio
                sh """
                    curl -s -X POST -H "Content-Type: application/json" \
                    -d '{
                        "status": "started",
                        "job": "${env.JOB_NAME}",
                        "build": "${env.BUILD_NUMBER}",
                        "timestamp":"${new Date().format('yyyy-MM-dd HH:mm:ss')}",
                        "message": "🚀 Build iniciado por Jenkins"
                    }' ${N8N_WEBHOOK}
                """
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

        stage('Build') {
            steps {
                echo '🔨 Construyendo aplicación Next.js...'
                sh 'npm run build'

                // Notificar progreso
                sh """
                    curl -s -X POST -H "Content-Type: application/json" \
                    -d '{
                        "status": "progress",
                        "job": "${env.JOB_NAME}",
                        "build": "${env.BUILD_NUMBER}",
                        "message": "⚙️ Build completado. Iniciando despliegue..."
                    }' ${N8N_WEBHOOK}
                """
            }
        }

        stage('Deploy') {
            steps {
                echo '🚀 Desplegando aplicación...'
                script {
                    sh '''
                        # Matar proceso anterior si existe
                        if [ -f /var/jenkins_home/workspace/vision-fe/app.pid ]; then
                            PID=$(cat /var/jenkins_home/workspace/vision-fe/app.pid)
                            kill -9 $PID || true
                            rm /var/jenkins_home/workspace/vision-fe/app.pid
                        fi
                        
                        # Iniciar nueva aplicación
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
                        // Usamos un timeout más corto y un '|| exit 1' para forzar el fallo si no responde
                        sh 'curl --connect-timeout 5 -f http://localhost:3000 || exit 1' 
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
                echo "⏱️  Duración: ${duration}"
                echo "🔢 Build: #${env.BUILD_NUMBER}"
                echo "🌐 URL: http://localhost:3000"
                echo "📅 ${new Date().format('dd/MM/yyyy HH:mm:ss')}"
                echo '========================================='

                // Notificar éxito
                sh """
                    curl -s -X POST -H "Content-Type: application/json" \
                    -d '{
                        "status": "success",
                        "job": "${env.JOB_NAME}",
                        "build": "${env.BUILD_NUMBER}",
                        "message": "✅ Build #${env.BUILD_NUMBER} completado correctamente en ${duration}"
                    }' ${N8N_WEBHOOK}
                """
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
                
                // Capturar el log de consola de Jenkins. Esto es más seguro.
                def consoleLog = currentBuild.rawBuild.getLog(50).join('\n')
                def sanitizedLog = consoleLog.replaceAll('"', "'").replaceAll("\\r?\\n", " \\n ")

                // Notificar fallo
                sh """
                    curl -s -X POST -H "Content-Type: application/json" \
                    -d '{
                        "status": "failed",
                        "job": "${env.JOB_NAME}",
                        "build": "${env.BUILD_NUMBER}",
                        "message": "❌ Build fallido en Jenkins. Error de curl: ${currentBuild.result}",
                        "log": "${sanitizedLog.length() > 500 ? sanitizedLog.substring(sanitizedLog.length() - 500) : sanitizedLog}"
                    }' ${N8N_WEBHOOK}
                """
            }
        }

        always {
            echo '🏁 Pipeline finalizado'
            echo "Estado final: ${currentBuild.result}"
        }
    }
}