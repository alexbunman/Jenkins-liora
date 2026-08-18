pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials') // ID de credentials Jenkins (user/pass DockerHub)
        DOCKERHUB_USER        = 'alexbunman'
        KUBECONFIG_CRED       = credentials('kubeconfig')             // ID de credentials Jenkins (fichier kubeconfig)

        MOVIE_IMAGE = "${DOCKERHUB_USER}/movie-service"
        CAST_IMAGE  = "${DOCKERHUB_USER}/cast-service"
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
    }

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    // Détermine l'environnement cible à partir de la branche
                    switch (env.BRANCH_NAME) {
                        case 'dev':
                            env.TARGET_NAMESPACE = 'dev'
                            break
                        case 'qa':
                            env.TARGET_NAMESPACE = 'qa'
                            break
                        case 'staging':
                            env.TARGET_NAMESPACE = 'staging'
                            break
                        case 'master':
                            env.TARGET_NAMESPACE = 'prod'
                            break
                        default:
                            env.TARGET_NAMESPACE = ''
                    }
                    echo "Branche: ${env.BRANCH_NAME} -> Namespace cible: ${env.TARGET_NAMESPACE}"
                }
            }
        }

        stage('Tests unitaires') {
            parallel {
                stage('movie-service') {
                    steps {
                        dir('movie-service') {
                            sh '''
                                python3.8 -m venv venv
                                . venv/bin/activate
                                pip install --upgrade pip
                                pip install -r requirements.txt
                                pytest || true
                            '''
                        }
                    }
                }
                stage('cast-service') {
                    steps {
                        dir('cast-service') {
                            sh '''
                                python3.8 -m venv venv
                                . venv/bin/activate
                                pip install --upgrade pip
                                pip install -r requirements.txt
                                pytest || true
                            '''
                        }
                    }
                }
            }
        }

        stage('Build images Docker') {
            steps {
                sh """
                    docker build -t ${MOVIE_IMAGE}:${IMAGE_TAG} ./movie-service
                    docker build -t ${CAST_IMAGE}:${IMAGE_TAG} ./cast-service
                    docker tag ${MOVIE_IMAGE}:${IMAGE_TAG} ${MOVIE_IMAGE}:latest
                    docker tag ${CAST_IMAGE}:${IMAGE_TAG} ${CAST_IMAGE}:latest
                """
            }
        }

        stage('Push DockerHub') {
            steps {
                sh """
                    echo \$DOCKERHUB_CREDENTIALS_PSW | docker login -u \$DOCKERHUB_CREDENTIALS_USR --password-stdin
                    docker push ${MOVIE_IMAGE}:${IMAGE_TAG}
                    docker push ${MOVIE_IMAGE}:latest
                    docker push ${CAST_IMAGE}:${IMAGE_TAG}
                    docker push ${CAST_IMAGE}:latest
                """
            }
        }

        stage('Déploiement Dev/QA/Staging') {
            when {
                expression { env.BRANCH_NAME in ['dev', 'qa', 'staging'] }
            }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh """
                        kubectl create namespace ${TARGET_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                        helm upgrade --install movie-service ./charts \\
                            --namespace ${TARGET_NAMESPACE} \\
                            --set fullnameOverride=movie-service \\
                            --set image.repository=${MOVIE_IMAGE} \\
                            --set image.tag=${IMAGE_TAG} \\
                            --set service.nodePort=30007

                        helm upgrade --install cast-service ./charts \\
                            --namespace ${TARGET_NAMESPACE} \\
                            --set fullnameOverride=cast-service \\
                            --set image.repository=${CAST_IMAGE} \\
                            --set image.tag=${IMAGE_TAG} \\
                            --set service.nodePort=30008
                    """
                }
            }
        }

        stage('Approbation manuelle Prod') {
            when {
                branch 'master'
            }
            steps {
                timeout(time: 24, unit: 'HOURS') {
                    input message: "Déployer la version ${IMAGE_TAG} en PRODUCTION ?", ok: 'Déployer'
                }
            }
        }

        stage('Déploiement Prod') {
            when {
                branch 'master'
            }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh """
                        kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -

                        helm upgrade --install movie-service ./charts \\
                            --namespace prod \\
                            --set fullnameOverride=movie-service \\
                            --set image.repository=${MOVIE_IMAGE} \\
                            --set image.tag=${IMAGE_TAG} \\
                            --set service.nodePort=30007

                        helm upgrade --install cast-service ./charts \\
                            --namespace prod \\
                            --set fullnameOverride=cast-service \\
                            --set image.repository=${CAST_IMAGE} \\
                            --set image.tag=${IMAGE_TAG} \\
                            --set service.nodePort=30008
                    """
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
        success {
            echo "Pipeline terminé avec succès pour la branche ${env.BRANCH_NAME}"
        }
        failure {
            echo "Échec du pipeline sur la branche ${env.BRANCH_NAME}"
        }
    }
}
