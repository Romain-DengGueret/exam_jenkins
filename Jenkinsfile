pipeline {
    environment {
        DOCKER_ID   = "rom1dgg"
        CAST_IMAGE  = "cast-service"
        MOVIE_IMAGE = "movie-service"
        DOCKER_TAG  = "v.${BUILD_ID}.0"   // tag incrémenté à chaque build
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
                    sleep 10
                    '''
                }
            }
        }

        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                    curl -sf localhost:8001/docs || (echo "Service KO" && exit 1)
                    docker rm -f jenkins-cast
                    '''
                }
            }
        }
    }
}
