pipeline {
    agent any

    environment {
        DOCKERHUB_CREDS = credentials('dockerhub')
        IMAGE_TAG       = "latest"
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'qa', 'staging', 'prod'],
            description: 'Environnement de déploiement'
        )
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Docker login') {
            steps {
                sh 'echo "$DOCKERHUB_CREDS_PSW" | docker login -u "$DOCKERHUB_CREDS_USR" --password-stdin'
            }
        }

        stage('Build & Push images') {
            steps {
                sh '''
                docker build -t rom1dgg/cast-service:${IMAGE_TAG} ./cast-service
                docker build -t rom1dgg/movie-service:${IMAGE_TAG} ./movie-service
                docker push rom1dgg/cast-service:${IMAGE_TAG}
                docker push rom1dgg/movie-service:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy databases') {
            steps {
                sh '''
                helm status cast-db -n ${ENVIRONMENT} >/dev/null 2>&1 || \
                  helm install cast-db bitnami/postgresql -n ${ENVIRONMENT} \
                    --set auth.username=cast_db_username \
                    --set auth.password=cast_db_password \
                    --set auth.database=cast_db_${ENVIRONMENT}
                helm status movie-db -n ${ENVIRONMENT} >/dev/null 2>&1 || \
                  helm install movie-db bitnami/postgresql -n ${ENVIRONMENT} \
                    --set auth.username=movie_db_username \
                    --set auth.password=movie_db_password \
                    --set auth.database=movie_db_${ENVIRONMENT}
                '''
            }
        }

        stage('Approval Production') {
            when { expression { params.ENVIRONMENT == 'prod' } }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: "Déployer ${env.JOB_NAME} en PRODUCTION ?", ok: 'Déployer'
                }
            }
        }

        stage('Deploy services') {
            when { branch 'master' }    // aucun déploiement hors master
            steps {
                sh 'helm upgrade --install cast-service ./charts -n ${ENVIRONMENT} -f ./charts/values-${ENVIRONMENT}.yaml'
                sh 'helm upgrade --install movie-service ./charts -n ${ENVIRONMENT} -f ./charts/values-movie-${ENVIRONMENT}.yaml'
            }
        }

        stage('Wait for rollout') {
            steps {
                sh 'kubectl rollout status deploy/cast-service-fastapiapp -n ${ENVIRONMENT} --timeout=180s'
                sh 'kubectl rollout status deploy/movie-service-fastapiapp -n ${ENVIRONMENT} --timeout=180s'
            }
        }

        stage('Smoke tests') {
            steps {
                sh '''
                export NODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")
                export CAST_PORT=$(kubectl get -n ${ENVIRONMENT} -o jsonpath="{.spec.ports[0].nodePort}" svc cast-service-fastapiapp)
                export MOVIE_PORT=$(kubectl get -n ${ENVIRONMENT} -o jsonpath="{.spec.ports[0].nodePort}" svc movie-service-fastapiapp)
                curl -sf http://$NODE_IP:$CAST_PORT/api/v1/checkapi || (echo "cast-service KO" && exit 1)
                curl -sf http://$NODE_IP:$MOVIE_PORT/api/v1/checkapi || (echo "movie-service KO" && exit 1)
                echo "Smoke tests OK"
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            cleanWs()
        }
        success   { echo "Déploiement ${params.ENVIRONMENT} réussi ✅" }
        failure   { echo "Pipeline FAILED sur ${params.ENVIRONMENT}" }
    }
}
