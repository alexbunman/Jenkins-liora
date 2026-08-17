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
