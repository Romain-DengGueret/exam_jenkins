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
    }
}
