pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKERHUB_USERNAME = "${DOCKERHUB_CREDENTIALS_USR}"
        DOCKERHUB_REPO = 'generador-claves'
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/${DOCKERHUB_REPO}"
        GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Determine Version') {
            steps {
                script {
                    def gitTag = sh(
                        script: "git describe --tags --exact-match HEAD 2>/dev/null || echo ''",
                        returnStdout: true
                    ).trim()

                    if (gitTag && gitTag.startsWith('v')) {
                        env.VERSION = gitTag.substring(1)
                        env.IS_RELEASE = 'true'
                    } else {
                        def packageVersion = sh(
                            script: "node -p \"require('./package.json').version\"",
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

        // 🔹 STAGE NUEVO: crear el archivo que falta
        stage('Crear .yarnrc.yml') {
            steps {
                sh 'echo "# temporal para evitar error" > .yarnrc.yml'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
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
                    sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"

                    sh """
                        docker push ${IMAGE_NAME}:${VERSION}
                        docker push ${IMAGE_NAME}:${GIT_COMMIT_SHORT}
                    """

                    if (env.IS_RELEASE == 'true') {
                        sh "docker push ${IMAGE_NAME}:latest"
                        echo "✅ Released version ${VERSION} as latest"
                    } else {
                        echo "⚠️ Development build - not updating 'latest' tag"
                    }

                    sh 'docker logout'
                }
            }
        }

        // 🔹 STAGE NUEVO: limpiar el archivo creado dinámicamente
        stage('Limpiar .yarnrc.yml') {
            steps {
                sh 'rm -f .yarnrc.yml'
            }
        }
    }

    post {
        always {
            cleanWs()
            script {
                sh """
                    CONTAINERS=\$(docker ps -aq --filter ancestor=${IMAGE_NAME} 2>/dev/null || echo "")
                    if [ ! -z "\$CONTAINERS" ]; then
                        echo "Limpiando contenedores: \$CONTAINERS"
                        docker stop \$CONTAINERS || true
                        docker rm \$CONTAINERS || true
                    else
                        echo "No hay contenedores que limpiar"
                    fi

                    docker rmi ${IMAGE_NAME}:${VERSION} || true
                    docker rmi ${IMAGE_NAME}:latest || true
                    docker rmi ${IMAGE_NAME}:${GIT_COMMIT_SHORT} || true
                    docker image prune -f || true
                """
            }
        }
        success {
            script {
                if (env.IS_RELEASE == 'true') {
                    echo """
                    🎉 ¡Release ${VERSION} publicado exitosamente!
                    docker pull ${IMAGE_NAME}:${VERSION}
                    docker pull ${IMAGE_NAME}:latest
                    docker run -d -p 3000:3000 ${IMAGE_NAME}:${VERSION}
                    """
                } else {
                    echo """
                    ✅ ¡Build de desarrollo completado!
                    docker pull ${IMAGE_NAME}:${VERSION}
                    docker run -d -p 3000:3000 ${IMAGE_NAME}:${VERSION}
                    docker pull ${IMAGE_NAME}:${GIT_COMMIT_SHORT}
                    """
                }
            }
        }
        failure {
            echo '❌ Error al construir o publicar la imagen'
        }
    }
}
