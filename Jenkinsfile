pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-v /var/run/docker.sock:/var/run/docker.sock --group-add $(stat -c %g /var/run/docker.sock)'
            reuseNode true
        }
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKERHUB_USERNAME = "${DOCKERHUB_CREDENTIALS_USR}"
        DOCKERHUB_REPO = 'generador-claves'
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/${DOCKERHUB_REPO}"
        YARN_VERSION = '3.6.1' // Versión estable de Yarn Berry
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup Yarn') {
            steps {
                script {
                    sh """
                    # Configurar Yarn Berry correctamente
                    mkdir -p .yarn/releases
                    echo "yarnPath: .yarn/releases/yarn-berry.cjs" > .yarnrc.yml
                    echo "nodeLinker: node-modules" >> .yarnrc.yml
                    echo "enableGlobalCache: true" >> .yarnrc.yml
                    
                    # Descargar Yarn Berry específico
                    curl -L https://github.com/yarnpkg/berry/releases/download/@yarnpkg/cli/${YARN_VERSION}/packages/yarnpkg-cli/bin/yarn.js -o .yarn/releases/yarn-berry.cjs
                    chmod +x .yarn/releases/yarn-berry.cjs
                    
                    # Verificar integridad
                    if ! grep -q "yarnPath" .yarnrc.yml; then
                        echo "ERROR: Invalid .yarnrc.yml format"
                        exit 1
                    fi
                    """
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh """
                # Instalar dependencias usando Yarn Berry
                yarn set version ${YARN_VERSION}
                yarn install --immutable
                """
            }
        }

        stage('Build Docker Image') {
            environment {
                DOCKER_BUILDKIT = '1'  # Habilita BuildKit
            }
            steps {
                script {
                    // Determinar versión automáticamente
                    def version = sh(
                        script: "grep '\"version\"' package.json | head -1 | awk -F: '{ print \$2 }' | sed 's/[\", ]//g'",
                        returnStdout: true
                    ).trim() + "-build.${BUILD_NUMBER}"

                    sh """
                    # Construir imagen con BuildKit
                    docker build \
                        --progress plain \
                        --no-cache \
                        -t ${IMAGE_NAME}:${version} \
                        -t ${IMAGE_NAME}:latest \
                        .
                    """
                }
            }
        }

        stage('Test Image') {
            steps {
                sh """
                # Prueba básica de la imagen
                docker run --rm ${IMAGE_NAME}:latest node --version
                docker run --rm ${IMAGE_NAME}:latest yarn --version
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )]) {
                        sh """
                        echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin
                        docker push ${IMAGE_NAME}:${version}
                        docker push ${IMAGE_NAME}:latest
                        docker logout
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
            script {
                sh """
                # Limpieza de contenedores e imágenes
                docker ps -aq | xargs -r docker rm -f || true
                docker images -q ${IMAGE_NAME} | xargs -r docker rmi -f || true
                docker system prune -f
                """
            }
        }
        success {
            echo "✅ Pipeline completado exitosamente!"
        }
        failure {
            echo "❌ Pipeline falló - revisar logs"
        }
    }
}
