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
        VERSION = '' // Variable para la versión de la imagen
        DOCKER_BUILDKIT = '1' // Habilita BuildKit
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
                    mkdir -p .yarn/releases
                    echo "yarnPath: .yarn/releases/yarn-berry.cjs" > .yarnrc.yml
                    echo "nodeLinker: node-modules" >> .yarnrc.yml
                    echo "enableGlobalCache: true" >> .yarnrc.yml

                    curl -L https://github.com/yarnpkg/berry/releases/download/@yarnpkg/cli/${YARN_VERSION}/packages/yarnpkg-cli/bin/yarn.js -o .yarn/releases/yarn-berry.cjs
                    chmod +x .yarn/releases/yarn-berry.cjs

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
                yarn set version ${YARN_VERSION}
                yarn install --immutable
                """
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.VERSION = sh(
                        script: "grep '\"version\"' package.json | head -1 | awk -F: '{ print \$2 }' | sed 's/[\", ]//g'",
                        returnStdout: true
                    ).trim() + "-build.${BUILD_NUMBER}"

                    sh """
                    docker build \
                        --progress plain \
                        --no-cache \
                        -t ${IMAGE_NAME}:${VERSION} \
                        -t ${IMAGE_NAME}:latest \
                        .
                    """
                }
            }
        }

        stage('Test Image') {
            steps {
                sh """
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
                        docker push ${IMAGE_NAME}:${VERSION}
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
