pipeline {
agent any
environment {
    PROJECT_ID = 'foreshy-auto'
    CLUSTER_NAME = 'k8s'
    LOCATION = 'asia-northeast3-a'
    CREDENTIALS_ID = 'gcp-service-account'
}
stages {
    stage("Checkout code") {
    steps {
        checkout scm
    }
    }
    stage("Build image") {
    steps {
        script {
        myapp = docker.build("dakyong/foreshy-app:${env.BUILD_ID}")
        }
    }
    }
    stage("Test image") {
    steps {
        script {
        withCredentials([string(credentialsId: 'db-password', variable: 'DB_PASSWORD')]) {
            myapp.withRun('-e DB_HOST=10.178.0.6 -e DB_USER=foreshy_user -e DB_PASSWORD=$DB_PASSWORD -e DB_NAME=foreshy_db') { c ->
            sh """
                APP_IP=\$(docker inspect -f '{{.NetworkSettings.IPAddress}}' ${c.id})
                for i in \$(seq 1 30); do
                curl -s -o /dev/null http://\$APP_IP:3000/auth/login && break
                sleep 3
                done
                newman run tests/foreshy.postman_collection.json \\
                -e tests/local.postman_environment.json \\
                --env-var baseUrl=http://\$APP_IP:3000/ \\
                --ignore-redirects || (docker logs ${c.id} --tail 50; exit 1)
            """
            }
        }
        }
    }
    }
    stage("Push image") {
    when {
        branch 'main'
    }
    steps {
        script {
        docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-token') {
            myapp.push("latest")
            myapp.push("${env.BUILD_ID}")
        }
        }
    }
    }
        stage('Deploy to GKE') {
    when {
        branch 'main'
    }
    steps {
        sh "sed -i 's/foreshy-app:latest/foreshy-app:${env.BUILD_ID}/g' deployment.yaml"
        withCredentials([
        string(credentialsId: 'db-password', variable: 'DB_PASSWORD'),
        string(credentialsId: 'session-secret', variable: 'SESSION_SECRET'),
        string(credentialsId: 'groq-api-key', variable: 'GROQ_API_KEY'),
        string(credentialsId: 'gemini-api-key', variable: 'GEMINI_API_KEY'),
        string(credentialsId: 'gemini-api-key-2', variable: 'GEMINI_API_KEY_2'),
        string(credentialsId: 'gemini-api-key-3', variable: 'GEMINI_API_KEY_3'),
        file(credentialsId: 'google-ocr-key', variable: 'OCR_KEY_FILE')
        ]) {
        sh '''
            sed -i "s|__DB_PASSWORD__|$(printf %s "$DB_PASSWORD" | base64 -w0)|" secret.yaml
            sed -i "s|__SESSION_SECRET__|$(printf %s "$SESSION_SECRET" | base64 -w0)|" secret.yaml
            sed -i "s|__GROQ_API_KEY__|$(printf %s "$GROQ_API_KEY" | base64 -w0)|" secret.yaml
            sed -i "s|__GEMINI_API_KEY__|$(printf %s "$GEMINI_API_KEY" | base64 -w0)|" secret.yaml
            sed -i "s|__GEMINI_API_KEY_2__|$(printf %s "$GEMINI_API_KEY_2" | base64 -w0)|" secret.yaml
            sed -i "s|__GEMINI_API_KEY_3__|$(printf %s "$GEMINI_API_KEY_3" | base64 -w0)|" secret.yaml
            sed -i "s|__GOOGLE_OCR_KEY__|$(base64 -w0 "$OCR_KEY_FILE")|" secret.yaml
        '''
        }
        step([$class: 'KubernetesEngineBuilder', projectId: env.PROJECT_ID, clusterName: env.CLUSTER_NAME, location: env.LOCATION, manifestPattern: 'secret.yaml', credentialsId: env.CREDENTIALS_ID, verifyDeployments: false])
        step([$class: 'KubernetesEngineBuilder', projectId: env.PROJECT_ID, clusterName: env.CLUSTER_NAME, location: env.LOCATION, manifestPattern: 'deployment.yaml', credentialsId: env.CREDENTIALS_ID, verifyDeployments: true])
        step([$class: 'KubernetesEngineBuilder', projectId: env.PROJECT_ID, clusterName: env.CLUSTER_NAME, location: env.LOCATION, manifestPattern: 'service.yaml', credentialsId: env.CREDENTIALS_ID, verifyDeployments: false])
    }
    }
}
}