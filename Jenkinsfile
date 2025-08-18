pipeline {
    agent any
    
    environment {
        // Configuración base
        IMAGE_NAME = "demo-ci-cd"
        IMAGE_TAG = "${BUILD_NUMBER}"
        CONTAINER_NAME = "demo-ci-cd"
        APP_PORT = "8080"
        
        // Configuración del registry (GitHub Packages)
        REGISTRY_URL = "ghcr.io"
        GITHUB_USER = "Yasir-hub1"  // ⚠️ CAMBIAR por tu usuario
        REGISTRY_IMAGE = "${REGISTRY_URL}/${GITHUB_USER}/${IMAGE_NAME}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '🔄 === CLONANDO REPOSITORIO ==='
                checkout scm
                
                echo '📋 Información del repositorio:'
                sh 'git log --oneline -5'
                sh 'ls -la'
            }
        }
        
        stage('Build & Test') {
            steps {
                echo '🔨 === COMPILANDO Y EJECUTANDO PRUEBAS ==='
                
                // Compilar y ejecutar tests
                sh 'mvn -B clean package'
                
                echo '📊 Resultados de la compilación:'
                sh 'ls -la target/'
                sh 'file target/*.jar'
                
                // Mostrar información del JAR generado
                script {
                    def jarFile = sh(returnStdout: true, script: 'ls target/*.jar').trim()
                    echo "✅ JAR generado: ${jarFile}"
                    sh "du -h ${jarFile}"
                }
            }
            post {
                always {
                    // Archivar reportes de pruebas
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo '🐳 === CONSTRUYENDO IMAGEN DOCKER ==='
                script {
                    // Construir imagen local
                    def localImage = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                    
                    // Tags adicionales
                    sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest"
                    sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY_IMAGE}:${IMAGE_TAG}"
                    sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY_IMAGE}:latest"
                    
                    echo "✅ Imágenes creadas:"
                    sh "docker images | grep ${IMAGE_NAME}"
                }
            }
        }
        
        stage('Package Artifacts') {
            steps {
                echo '📦 === EMPAQUETANDO ARTEFACTOS ==='
                script {
                    // Crear archivo comprimido de la imagen Docker
                    echo 'Exportando imagen Docker...'
                    sh "docker save ${IMAGE_NAME}:${IMAGE_TAG} | gzip > ${IMAGE_NAME}-${IMAGE_TAG}.tar.gz"
                    
                    // Información de artefactos
                    sh "ls -lh ${IMAGE_NAME}-${IMAGE_TAG}.tar.gz"
                    sh "ls -lh target/*.jar"
                    
                    echo '📁 Archivando artefactos en Jenkins...'
                    archiveArtifacts artifacts: 'target/*.jar, *.tar.gz', fingerprint: true
                }
            }
        }
        
        stage('Publish to Registry') {
            steps {
                echo '🚀 === PUBLICANDO EN REGISTRY REMOTO ==='
                script {
                    // Hacer push a GitHub Container Registry
                    docker.withRegistry("https://${REGISTRY_URL}", 'github-packages') {
                        echo "Haciendo push a ${REGISTRY_URL}..."
                        
                        def image = docker.image("${REGISTRY_IMAGE}:${IMAGE_TAG}")
                        image.push()
                        image.push('latest')
                        
                        echo "✅ Imagen publicada en: ${REGISTRY_IMAGE}:${IMAGE_TAG}"
                        echo "✅ Imagen publicada en: ${REGISTRY_IMAGE}:latest"
                    }
                    
                    // También publicar JAR si tienes Nexus/Artifactory configurado
                    // mvn deploy (requiere configuración en pom.xml)
                }
            }
        }
        
        stage('Deploy to Test Environment') {
            steps {
                echo '🏗️ === DESPLEGANDO EN ENTORNO DE PRUEBAS ==='
                script {
                    // Limpiar contenedor anterior
                    sh """
                        echo 'Limpiando contenedor anterior...'
                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true
                    """
                    
                    // Desplegar nuevo contenedor
                    sh """
                        echo 'Desplegando nueva versión...'
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${APP_PORT}:8080 \
                            --restart unless-stopped \
                            ${IMAGE_NAME}:${IMAGE_TAG}
                    """
                    
                    // Esperar inicialización
                    echo '⏳ Esperando inicialización de la aplicación...'
                    sh 'sleep 20'
                }
            }
        }
        
        stage('Validate Deployment') {
            steps {
                echo '✅ === VALIDANDO DESPLIEGUE ==='
                script {
                    // Verificar estado del contenedor
                    def containerStatus = sh(
                        script: "docker ps --filter name=${CONTAINER_NAME} --format '{{.Status}}'",
                        returnStdout: true
                    ).trim()
                    
                    echo "📊 Estado del contenedor: ${containerStatus}"
                    
                    if (containerStatus.contains('Up')) {
                        echo "✅ Contenedor ejecutándose correctamente"
                        
                        // Mostrar información del contenedor
                        sh "docker ps --filter name=${CONTAINER_NAME}"
                        sh "docker port ${CONTAINER_NAME}"
                        
                        // Probar conectividad
                        echo '🔍 Probando conectividad...'
                        sh '''
                            for i in {1..10}; do
                                echo "Intento $i de conectividad..."
                                if curl -f -s -m 5 http://localhost:8080/ > /dev/null; then
                                    echo "✅ Aplicación responde correctamente"
                                    echo "📄 Respuesta de la aplicación:"
                                    curl -s http://localhost:8080/ | head -20
                                    break
                                else
                                    echo "⏳ Esperando... ($i/10)"
                                    sleep 5
                                fi
                                
                                if [ $i -eq 10 ]; then
                                    echo "❌ Aplicación no responde después de 10 intentos"
                                    exit 1
                                fi
                            done
                        '''
                        
                    } else {
                        error("❌ El contenedor no está ejecutándose")
                    }
                }
            }
        }
        
        stage('Integration Tests') {
            steps {
                echo '🧪 === PRUEBAS DE INTEGRACIÓN ==='
                script {
                    // Pruebas de endpoints
                    sh '''
                        echo "=== PRUEBAS DE ENDPOINTS ==="
                        
                        echo "1. Prueba de endpoint raíz:"
                        HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/)
                        echo "Código HTTP: $HTTP_CODE"
                        
                        if [ "$HTTP_CODE" = "200" ]; then
                            echo "✅ Endpoint raíz OK"
                        else
                            echo "⚠️ Endpoint raíz retorna: $HTTP_CODE"
                        fi
                        
                        echo "2. Prueba de endpoint de salud:"
                        curl -s http://localhost:8080/actuator/health || echo "Actuator no disponible"
                        
                        echo "3. Información del sistema:"
                        curl -s http://localhost:8080/actuator/info || echo "Info endpoint no disponible"
                    '''
                    
                    // Verificar logs de la aplicación
                    echo '📋 Logs recientes de la aplicación:'
                    sh "docker logs ${CONTAINER_NAME} --tail 50"
                }
            }
        }
        
        stage('Generate Report') {
            steps {
                echo '📊 === GENERANDO REPORTE FINAL ==='
                script {
                    // Crear reporte de despliegue
                    sh '''
                        echo "=== REPORTE DE DESPLIEGUE ===" > deployment-report.txt
                        echo "Fecha: $(date)" >> deployment-report.txt
                        echo "Build: ${BUILD_NUMBER}" >> deployment-report.txt
                        echo "Imagen: ${IMAGE_NAME}:${IMAGE_TAG}" >> deployment-report.txt
                        echo "Registry: ${REGISTRY_IMAGE}" >> deployment-report.txt
                        echo "Puerto: ${APP_PORT}" >> deployment-report.txt
                        echo "" >> deployment-report.txt
                        echo "=== ESTADO DEL CONTENEDOR ===" >> deployment-report.txt
                        docker ps --filter name=${CONTAINER_NAME} >> deployment-report.txt
                        echo "" >> deployment-report.txt
                        echo "=== ARTEFACTOS GENERADOS ===" >> deployment-report.txt
                        ls -la target/*.jar >> deployment-report.txt
                        ls -la *.tar.gz >> deployment-report.txt
                    '''
                    
                    // Archivar reporte
                    archiveArtifacts artifacts: 'deployment-report.txt', fingerprint: true
                    
                    // Mostrar reporte
                    echo '📄 Contenido del reporte:'
                    sh 'cat deployment-report.txt'
                }
            }
        }
    }
    
    post {
        always {
            echo '🧹 === TAREAS FINALES ==='
            
            // Limpiar imágenes antiguas (mantener últimas 3)
            sh '''
                echo "Limpiando imágenes antiguas..."
                docker images ${IMAGE_NAME} --format "table {{.Repository}}:{{.Tag}}\t{{.CreatedAt}}" || true
                
                # Limpiar sistema Docker (cuidadosamente)
                docker system prune -f --filter "until=24h" || true
            '''
            
            // Información final
            sh '''
                echo "=== INFORMACIÓN FINAL ==="
                echo "Contenedores activos:"
                docker ps
                echo "Imágenes disponibles:"
                docker images | grep ${IMAGE_NAME} || true
                echo "Espacio en disco:"
                df -h
            '''
        }
        success {
            echo '''
            🎉 ===============================================
               ¡PIPELINE COMPLETADO EXITOSAMENTE!
            ===============================================
            '''
            echo "🌐 Aplicación: http://localhost:${APP_PORT}/"
            echo "📦 Artefactos: Archivados en Jenkins"
            echo "🐳 Registry: ${REGISTRY_IMAGE}:${IMAGE_TAG}"
            echo "📊 Build: #${BUILD_NUMBER}"
            
            // Enviar notificación (opcional)
            // slackSend color: 'good', message: "✅ Deploy exitoso: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        failure {
            echo '❌ ==============================================='
            echo '     PIPELINE FALLÓ - INFORMACIÓN DE DEBUG'
            echo '==============================================='
            
            script {
                sh '''
                    echo "=== CONTENEDORES ==="
                    docker ps -a
                    
                    echo "=== IMÁGENES ==="
                    docker images
                    
                    echo "=== LOGS DEL CONTENEDOR ==="
                    docker logs ${CONTAINER_NAME} --tail 100 || echo "Sin logs disponibles"
                    
                    echo "=== ESPACIO EN DISCO ==="
                    df -h
                    
                    echo "=== PROCESOS ==="
                    ps aux | grep java || true
                '''
            }
        }
        cleanup {
            echo '🗑️ Limpieza final completada'
        }
    }
}