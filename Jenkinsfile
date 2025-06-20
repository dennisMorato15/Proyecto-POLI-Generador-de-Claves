pipeline {
    agent any

    options {
        skipDefaultCheckout(true)  // Evitar checkout automático
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        // Configuración de DockerHub
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKERHUB_USERNAME = "${DOCKERHUB_CREDENTIALS_USR}"
        DOCKERHUB_REPO = 'generador-claves'

        // Etiquetas de la imagen con versionado semántico
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/${DOCKERHUB_REPO}"
    }

    stages {
        stage('Cleanup Workspace') {
            steps {
                script {
                    echo "🧹 Limpieza agresiva del workspace..."
                    
                    // Cambiar permisos y eliminar archivos problemáticos
                    sh '''
                        # Ir al directorio de trabajo
                        cd /var/jenkins_home/workspace/
                        
                        # Intentar cambiar permisos de archivos problemáticos
                        if [ -d "CI-Generador-Claves" ]; then
                            echo "Cambiando permisos de archivos problemáticos..."
                            find CI-Generador-Claves -name ".yarn" -type d -exec chmod -R 777 {} + || true
                            find CI-Generador-Claves -path "*/.yarn/*" -exec chmod 777 {} + || true
                            
                            echo "Eliminando directorio existente..."
                            rm -rf CI-Generador-Claves || true
                        fi
                        
                        echo "Creando directorio limpio..."
                        mkdir -p CI-Generador-Claves
                        cd CI-Generador-Claves
                        pwd
                    '''
                }
            }
        }

        stage('Manual Checkout') {
            steps {
                script {
                    echo "📥 Descarga manual del código..."
                    sh '''
                        # Ir al directorio de trabajo
                        cd /var/jenkins_home/workspace/CI-Generador-Claves
                        
                        # Clonar el repositorio
                        git clone https://github.com/Eziuz/Proyecto-POLI-Generador-de-Claves.git .
                        
                        # Verificar que se clonó correctamente
                        ls -la
                        
                        # Obtener información del commit
                        git log --oneline -1
                    '''
                    
                    // Establecer variables de Git después del checkout manual
                    env.GIT_COMMIT_SHORT = sh(script: 'cd /var/jenkins_home/workspace/CI-Generador-Claves && git rev-parse --short HEAD', returnStdout: true).trim()
                    echo "🔖 Git commit: ${env.GIT_COMMIT_SHORT}"
                }
            }
        }

        stage('Determine Version') {
            steps {
                script {
                    // Cambiar al directorio correcto
                    sh 'cd /var/jenkins_home/workspace/CI-Generador-Claves'
                    
                    // Intentar obtener la versión desde Git tags
                    def gitTag = sh(
                        script: "cd /var/jenkins_home/workspace/CI-Generador-Claves && git describe --tags --exact-match HEAD 2>/dev/null || echo ''",
                        returnStdout: true
                    ).trim()

                    if (gitTag && gitTag.startsWith('v')) {
                        // Si hay un tag que empiece con 'v', usarlo como versión
                        env.VERSION = gitTag.substring(1) // Remover la 'v' del inicio
                        env.IS_RELEASE = 'true'
                    } else {
                        // Si no hay tag, usar versión desde package.json + build number
                        def packageVersion = sh(
                            script: "cd /var/jenkins_home/workspace/CI-Generador-Claves && node -p \"require('./package.json').version\"",
                            returnStdout: true
                        ).trim()
                        env.VERSION = "${packageVersion}-build.${BUILD_NUMBER}"
                        env.IS_RELEASE = 'false'
                    }

                    echo "🏷️ Version determined: ${env.VERSION}"
                    echo "📦 Is release: ${env.IS_RELEASE}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Cambiar al directorio correcto
                    sh 'cd /var/jenkins_home/workspace/CI-Generador-Claves'
                    
                    // Construir la imagen Docker con versionado semántico
                    sh """
                        cd /var/jenkins_home/workspace/CI-Generador-Claves
                        docker build -t ${IMAGE_NAME}:${VERSION} \
                                     -t ${IMAGE_NAME}:latest \
                                     -t ${IMAGE_NAME}:${GIT_COMMIT_SHORT} .
                    """
                }
            }
        }

        stage('Verify Image') {
            steps {
                script {
                    sh """
                        echo "Verificando que la imagen existe..."
                        docker images ${IMAGE_NAME}:${VERSION}

                        echo "Verificando estructura de la imagen..."
                        docker inspect ${IMAGE_NAME}:${VERSION} > /dev/null

                        echo "Verificando configuración de la imagen..."
                        docker inspect ${IMAGE_NAME}:${VERSION} | grep -E '"User"|"Entrypoint"|"Cmd"|"WorkingDir"' || true

                        echo "Imagen verificada exitosamente"
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    // Iniciar sesión en DockerHub
                    sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"

                    // Publicar las imágenes en DockerHub
                    sh """
                        docker push ${IMAGE_NAME}:${VERSION}
                        docker push ${IMAGE_NAME}:${GIT_COMMIT_SHORT}
                    """

                    // Solo pushear 'latest' si es un release oficial
                    if (env.IS_RELEASE == 'true') {
                        sh "docker push ${IMAGE_NAME}:latest"
                        echo "✅ Released version ${VERSION} as latest"
                    } else {
                        echo "⚠️ Development build - not updating 'latest' tag"
                    }

                    // Cerrar sesión de DockerHub
                    sh 'docker logout'
                }
            }
        }
    }

    post {
        always {
            script {
                echo "🧹 Limpieza final..."
                
                // Solo limpiar Docker si las variables existen
                if (env.IMAGE_NAME && env.VERSION) {
                    sh """
                        # Obtener contenedores que usan nuestras imágenes
                        CONTAINERS=\$(docker ps -aq --filter ancestor=${IMAGE_NAME} 2>/dev/null || echo "")
                        if [ ! -z "\$CONTAINERS" ]; then
                            echo "Limpiando contenedores: \$CONTAINERS"
                            docker stop \$CONTAINERS || true
                            docker rm \$CONTAINERS || true
                        else
                            echo "No hay contenedores que limpiar"
                        fi

                        # Eliminar las imágenes si existen
                        docker rmi ${IMAGE_NAME}:${VERSION} || true
                        docker rmi ${IMAGE_NAME}:latest || true
                    """
                    
                    // Solo si GIT_COMMIT_SHORT existe
                    if (env.GIT_COMMIT_SHORT) {
                        sh "docker rmi ${IMAGE_NAME}:${GIT_COMMIT_SHORT} || true"
                    }
                    
                    sh "docker image prune -f || true"
                } else {
                    echo "Variables de imagen no disponibles, saltando limpieza de Docker"
                }
                
                // Limpiar workspace
                try {
                    sh '''
                        cd /var/jenkins_home/workspace/
                        if [ -d "CI-Generador-Claves" ]; then
                            # Cambiar permisos antes de eliminar
                            find CI-Generador-Claves -name ".yarn" -type d -exec chmod -R 777 {} + || true
                            find CI-Generador-Claves -path "*/.yarn/*" -exec chmod 777 {} + || true
                            # Eliminar directorio
                            rm -rf CI-Generador-Claves || true
                        fi
                    '''
                } catch (Exception e) {
                    echo "Warning: No se pudo limpiar workspace: ${e.getMessage()}"
                }
            }
        }
        success {
            script {
                if (env.IS_RELEASE == 'true') {
                    echo """
                    🎉 ¡Release ${VERSION} publicado exitosamente!

                    Para usar esta versión:
                    docker pull ${IMAGE_NAME}:${VERSION}
                    docker pull ${IMAGE_NAME}:latest
                    docker run -d -p 3000:3000 ${IMAGE_NAME}:${VERSION}
                    """
                } else {
                    echo """
                    ✅ ¡Build de desarrollo completado!

                    Para usar esta versión:
                    docker pull ${IMAGE_NAME}:${VERSION}
                    docker run -d -p 3000:3000 ${IMAGE_NAME}:${VERSION}
                    """
                    
                    if (env.GIT_COMMIT_SHORT) {
                        echo """
                        También disponible por commit:
                        docker pull ${IMAGE_NAME}:${GIT_COMMIT_SHORT}
                        """
                    }
                }
            }
        }
        failure {
            echo '❌ Error al construir o publicar la imagen'
        }
    }
}
