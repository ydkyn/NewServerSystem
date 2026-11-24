def app

pipeline {
    agent any

    environment {
        IMAGE_NAME     = 'dakyong/foreshy-app'
        TEST_NAME      = "foreshy-test-${env.BUILD_NUMBER}"
        PROJECT_ID     = 'foreshy-auto'
        CLUSTER_NAME   = 'k8s'
        LOCATION       = 'asia-northeast3-a'
        GKE_CREDENTIAL = 'gcp-service-account'
    }

    stages {

        stage('Build image') {
            steps {
                script {
                    app = docker.build("${IMAGE_NAME}:${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Test') {
            steps {
                withCredentials([string(credentialsId: 'db-password', variable: 'DB_PASSWORD')]) {
                    sh '''
                    docker network create $TEST_NAME
                    docker run -d --name $TEST_NAME --network $TEST_NAME \
                        -e NODE_ENV=production -e PORT=3000 \
                        -e DB_HOST=10.178.0.6 -e DB_PORT=3306 \
                        -e DB_USER=foreshy_user -e DB_PASSWORD="$DB_PASSWORD" -e DB_NAME=foreshy_db \
                        $IMAGE_NAME:$BUILD_NUMBER
                    docker network connect $TEST_NAME $(hostname)
                    '''
                }

                sh '''
                for i in $(seq 1 30); do
                    curl -s -o /dev/null http://$TEST_NAME:3000/auth/login && break
                    sleep 3
                done
                '''

                sh '''
                newman run tests/foreshy.postman_collection.json \
                    -e tests/local.postman_environment.json \
                    --env-var "baseUrl=http://$TEST_NAME:3000/" \
                    --ignore-redirects
                '''
            }
            post {
                always {
                    sh '''
                    docker logs $TEST_NAME --tail 50 || true
                    docker network disconnect $TEST_NAME $(hostname) || true
                    docker rm -f $TEST_NAME || true
                    docker network rm $TEST_NAME || true
                    '''
                }
            }
        }

        stage('Push image') {
            when { branch 'main' }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-token') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push('latest')
                    }
                }
            }
        }

        stage('Deploy to GKE') {
            when { branch 'main' }
            steps {
                withCredentials([string(credentialsId: 'db-password', variable: 'DB_PASSWORD')]) {
                    sh '''
                    sed -i "s|dakyong/foreshy-app:latest|dakyong/foreshy-app:$BUILD_NUMBER|" deployment.yaml
                    sed -i "s|__DB_PASSWORD__|$DB_PASSWORD|" deployment.yaml
                    '''
                }
                step([$class: 'KubernetesEngineBuilder',
                        projectId: env.PROJECT_ID,
                        clusterName: env.CLUSTER_NAME,
                        location: env.LOCATION,
                        manifestPattern: 'deployment.yaml',
                        credentialsId: env.GKE_CREDENTIAL,
                        verifyDeployments: true])
                step([$class: 'KubernetesEngineBuilder',
                        projectId: env.PROJECT_ID,
                        clusterName: env.CLUSTER_NAME,
                        location: env.LOCATION,
                        manifestPattern: 'service.yaml',
                        credentialsId: env.GKE_CREDENTIAL,
                        verifyDeployments: false])
            }
        }
    }
}
