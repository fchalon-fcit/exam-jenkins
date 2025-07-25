pipeline {
    agent any

    environment {
        DOCKER_ID = "fbst"
        DOCKER_IMAGE_CAST = "exam-jenkins-cast"
        DOCKER_IMAGE_MOVIE = "exam-jenkins-movie"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }

    stages {
        stage('Docker Build') {
            steps {
                script {
                    sh '''
                        docker rm -f exam-app || true
                        docker-compose -f docker-compose.yml build

                        docker tag exam-jenkins-cast_service $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                        docker tag exam-jenkins-movie_service $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                        docker-compose -f docker-compose.yml up -d
                        echo "Attente du démarrage des services..."
                        sleep 10

                        echo "Test de cast_service..."
                        curl --retry 5 --retry-delay 3 --fail http://localhost:8888/api/v1/casts/docs

                        echo "Test de movie_service..."
                        curl --retry 5 --retry-delay 3 --fail http://localhost:8888/api/v1/movies/docs

                        docker-compose down
                    '''
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_ID --password-stdin
                        docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                        docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Déploiement en dev') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                        rm -rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config

                        cp charts/values.yaml values.yml
                        sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml

                        helm upgrade --install app-dev charts/ --values=values.yml --namespace dev --create-namespace
                    '''
                }
            }
        }

        stage('Déploiement en qa') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                        rm -rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config

                        cp charts/values.yaml values.yml
                        sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml

                        helm upgrade --install app-qa charts/ --values=values.yml --namespace qa --create-namespace
                    '''
                }
            }
        }

        stage('Déploiement en staging') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                        rm -rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config

                        cp charts/values.yaml values.yml
                        sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml

                        helm upgrade --install app-staging charts/ --values=values.yml --namespace staging --create-namespace
                    '''
                }
            }
        }

        stage('Déploiement en prod') {

            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Souhaitez-vous déployer en production ?', ok: 'Déployer'
                }

                script {
                    sh '''
                        rm -rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config

                        cp charts/values.yaml values.yml
                        sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml

                        helm upgrade --install app-prod charts/ --values=values.yml --namespace prod --create-namespace
                    '''
                }
            }
        }
    }

}

