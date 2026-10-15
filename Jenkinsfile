pipeline {
    environment {
        DOCKER_ID   = "rom1dgg"
        CAST_IMAGE  = "cast-service"
        MOVIE_IMAGE = "movie-service"
        DOCKER_TAG  = "v.${BUILD_ID}.0"
    }
    agent any

    stages {

        stage('Docker Build') {
            steps {
                script {
                    sh '''
                    docker build -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG ./cast-service
                    docker build -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG ./movie-service
                    '''
                }
            }
        }

        stage('Docker Run') {
            steps {
                script {
                    sh '''
                    docker rm -f jenkins-cast || true
                    docker run -d -p 8001:8000 --name jenkins-cast \
                      -e DATABASE_URI="postgresql://cast_db_username:cast_db_password@cast-db:5432/cast_db_dev" \
                      $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
                    sleep 5
                    docker rm -f jenkins-cast
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
                    echo "$DOCKER_PASS" | docker login -u $DOCKER_ID --password-stdin
                    docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
                    docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
                    docker logout
                    '''
                }
            }
        }

        stage('Deploiement en dev') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                    rm -Rf ~/.kube/
                    mkdir ~/.kube/
                    cat $KUBECONFIG > ~/.kube/config

                    cp charts/values-dev.yaml cast-values.yml
                    sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" cast-values.yml
                    helm upgrade --install cast-service ./charts -n dev -f cast-values.yml

                    cp charts/values-movie-dev.yaml movie-values.yml
                    sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" movie-values.yml
                    helm upgrade --install movie-service ./charts -n dev -f movie-values.yml
                    '''
                }
            }
        }

        stage('Deploiement en qa') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                    rm -Rf ~/.kube/
                    mkdir ~/.kube/
                    cat $KUBECONFIG > ~/.kube/config

                    cp charts/values-qa.yaml cast-values.yml
                    sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" cast-values.yml
                    helm upgrade --install cast-service ./charts -n qa -f cast-values.yml

                    cp charts/values-movie-qa.yaml movie-values.yml
                    sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" movie-values.yml
                    helm upgrade --install movie-service ./charts -n qa -f movie-values.yml
                    '''
                }
            }
        }

        stage('Deploiement en staging') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                    rm -Rf ~/.kube/
                    mkdir ~/.kube/
                    cat $KUBECONFIG > ~/.kube/config

                    cp charts/values-staging.yaml cast-values.yml
                    sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" cast-values.yml
                    helm upgrade --install cast-service ./charts -n staging -f cast-values.yml

                    cp charts/values-movie-staging.yaml movie-values.yml
                    sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" movie-values.yml
                    helm upgrade --install movie-service ./charts -n staging -f movie-values.yml
                    '''
                }
            }
        }

        stage('Deploiement en prod') {
            when { branch 'master' }   // déploiement prod uniquement depuis master
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                // Validation manuelle obligatoire avant la prod (timeout 15 min)
                timeout(time: 15, unit: "MINUTES") {
                    input message: "Déployer en PRODUCTION ?", ok: 'Oui, déployer'
                }
                script {
                    sh '''
                    rm -Rf ~/.kube/
                    mkdir ~/.kube/
                    cat $KUBECONFIG > ~/.kube/config

                    cp charts/values-prod.yaml cast-values.yml
                    sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" cast-values.yml
                    helm upgrade --install cast-service ./charts -n prod -f cast-values.yml

                    cp charts/values-movie-prod.yaml movie-values.yml
                    sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" movie-values.yml
                    helm upgrade --install movie-service ./charts -n prod -f movie-values.yml
                    '''
                }
            }
        }
    }
}
