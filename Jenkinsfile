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

        stage('Docker Run') {
            steps {
                script {
                    sh '''
                        docker run -d -p 8000:8000 --name exam-app $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                        sleep 10
                    '''
                }
            }
        }

        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                        curl -f http://localhost:8000/ || exit 1
                        docker stop exam-app
                        docker rm exam-app
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
            when {
                branch 'master'
            }
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

