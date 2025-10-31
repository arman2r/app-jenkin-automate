tools {
    nodejs 'NodeJS 22.x'
}

environment {
    APP_NAME = 'my-app-nextjs'
    APP_PORT = '3000'
    N8N_WEBHOOK = 'http://host.docker.internal:5678/webhook/jenkins-events'
}

triggers {
    pollSCM('H/5 * * * *')
}

stages {

    stage('Inicio') {
        steps {
            script {
                // Guardamos la hora de inicio en formato ISO8601
                env.START_TIME = new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'")
                currentBuild.description = "Build #${env.BUILD_NUMBER} - Iniciado"

                echo '🚀 Iniciando pipeline de deployment...'

                // Notificar inicio a n8n
                sh """
                    curl -s -X POST -H "Content-Type: application/json" \
                    -d '{
                        "status": "started",
                        "job": "${env.JOB_NAME}",
                        "build": "${env.BUILD_NUMBER}",
                        "timestamp": "${env.START_TIME}",
                        "message": "🚀 Build iniciado por Jenkins"
                    }' ${N8N_WEBHOOK}
                """
            }
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

            // Notificar progreso a n8n
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
                    mkdir -p /var/jenkins_home/workspace/vision-fe
                    nohup npm run start > /var/jenkins_home/workspace/vision-fe/app.log 2>&1 &
                    echo $! > /var/jenkins_home/workspace/vision-fe/app.pid
                    echo "Aplicación iniciada con PID: $(cat /var/jenkins_home/workspace/vision-fe/app.pid)"
                '''
                sleep(time: 10, unit: 'SECONDS')
                echo "✅ Aplicación desplegada en http://localhost:${APP_PORT}"
            }
        }
    }

    stage('Health Check') {
        steps {
            echo '❤️ Verificando salud de la aplicación...'
            script {
                retry(3) {
                    sleep(time: 2, unit: 'SECONDS')
                    sh 'curl -f http://localhost:3000 || exit 1'
                    echo '✅ Aplicación respondiendo correctamente'
                }
            }
        }
    }
}

post {
    success {
        script {
            def endTime = new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'")
            def duration = currentBuild.durationString.replace(' and counting', '')

            echo '========================================='
            echo '✅ DEPLOYMENT EXITOSO'
            echo '========================================='
            echo "⏱️  Duración: ${duration}"
            echo "🔢 Build: #${env.BUILD_NUMBER}"
            echo "🌐 URL: http://localhost:${APP_PORT}"
            echo "📅 ${endTime}"
            echo '========================================='

            // Notificar éxito con tiempo total
            sh """
                curl -s -X POST -H "Content-Type: application/json" \
                -d '{
                    "status": "success",
                    "job": "${env.JOB_NAME}",
                    "build": "${env.BUILD_NUMBER}",
                    "start_time": "${env.START_TIME}",
                    "timestamp": "${endTime}",
                    "message": "✅ Build #${env.BUILD_NUMBER} completado correctamente en ${duration}"
                }' ${N8N_WEBHOOK}
            """
        }
    }

    failure {
        script {
            def endTime = new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'")

            echo '========================================='
            echo '❌ DEPLOYMENT FALLIDO'
            echo '========================================='
            echo "🔢 Build: #${env.BUILD_NUMBER}"
            echo "📅 ${endTime}"
            echo '========================================='

            // Captura últimos logs relevantes
            def logSnippet = sh(returnStdout: true, script: 'tail -n 200 /var/jenkins_home/workspace/vision-fe/app.log || echo "No se pudieron leer los logs"').trim()
            def sanitizedLog = logSnippet.replaceAll('"', "'").replaceAll("\\r?\\n", " \\n ")

            // Notificar fallo con log y timestamps
            sh """
                curl -s -X POST -H "Content-Type: application/json" \
                -d '{
                    "status": "failed",
                    "job": "${env.JOB_NAME}",
                    "build": "${env.BUILD_NUMBER}",
                    "start_time": "${env.START_TIME}",
                    "timestamp": "${endTime}",
                    "message": "❌ Build fallido en Jenkins",
                    "log": "${sanitizedLog}"
                }' ${N8N_WEBHOOK}
            """
        }
    }

    always {
        echo '🏁 Pipeline finalizado'
        echo "Estado final: ${currentBuild.result}"
    }
}
