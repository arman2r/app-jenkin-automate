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
        stage('Inicio & Checkout') {
            steps {
                script {
                    // [RE-ACTIVACIÓN] Reintroducimos la carga de la herramienta NodeJS para corregir "npm: not found".
                    // **Asegúrate que el nombre 'NodeJS 22.x' COINCIDA EXACTAMENTE con tu configuración en Manage Jenkins > Tools.**
                    try {
                        def nodeHome = tool name: 'NodeJS 22.x', type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
                        env.PATH = "${nodeHome}/bin:${env.PATH}"
                        echo "NodeJS configurado usando la herramienta: NodeJS 22.x"
                    } catch (e) {
                        echo "⚠️ Advertencia: No se pudo cargar la herramienta NodeJS. 'npm ci' podría fallar si Node no está en el PATH del agente."
                        // Continuamos sin la herramienta, el fallo de 'npm: not found' se producirá en la siguiente etapa.
                    }

                    currentBuild.description = "Build #${env.BUILD_NUMBER} - Iniciado"
                }

                echo '🚀 Iniciando pipeline de deployment...'

                // Notificar inicio - Usamos catchError para ignorar fallos de red.
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
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

                echo '📦 Clonando repositorio...'
                checkout scm // Checkout explícito

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

                // Notificar progreso - Usamos catchError para ignorar fallos de red.
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
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
                        # NOTA: Asegúrate que el directorio de trabajo tenga los archivos de build de Next.js
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

                // Notificar éxito - Usamos catchError para ignorar fallos de red.
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
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
        }

        failure {
            script {
                echo '========================================='
                echo '❌ DEPLOYMENT FALLIDO'
                echo '========================================='
                echo "🔢 Build: #${env.BUILD_NUMBER}"
                echo "📅 ${new Date().format('dd/MM/yyyy HH:mm:ss')}"
                echo '========================================='
                
                def failureMessage = "❌ Build fallido en Jenkins. Etapa de fallo: ${currentBuild.result}"

                // Notificar fallo - Usamos catchError para ignorar fallos de red.
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh """
                        curl -s -X POST -H "Content-Type: application/json" \
                        -d '{
                            "status": "failed",
                            "job": "${env.JOB_NAME}",
                            "build": "${env.BUILD_NUMBER}",
                            "message": "${failureMessage}",
                            "log": "Revisar la consola de Jenkins para más detalles."
                        }' ${N8N_WEBHOOK}
                    """
                }
            }
        }

        always {
            echo '🏁 Pipeline finalizado'
            echo "Estado final: ${currentBuild.result}"
        }
    }
}
