pipeline {
    agent { label 'maven-agent' }

    environment {
        APP_SERVER_IP   = '10.2.1.186'
        APP_SERVER_USER = 'ubuntu'
        APP_SERVER_DIR  = '/home/ubuntu/app'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean test'
            }
        }

        stage('Security Scan') {
            steps {
                sh 'trivy fs --exit-code 1 --severity HIGH,CRITICAL .'
            }
        }

        stage('Package') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

       stage('Deploy to Application Server') {
    when {
        anyOf {
            branch 'main'
            branch 'master'
        }
    }
    steps {
        sshagent(['app-server-ssh']) {
            sh """
                ssh -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} "mkdir -p ${APP_SERVER_DIR}"
                scp -o StrictHostKeyChecking=no target/*.jar ${APP_SERVER_USER}@${APP_SERVER_IP}:${APP_SERVER_DIR}/app.jar
                ssh -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} "pkill -f app.jar || true; nohup java -jar ${APP_SERVER_DIR}/app.jar > ${APP_SERVER_DIR}/app.log 2>&1 < /dev/null &"
            """
        }
    }
}
    }

    post {
        always {
            echo 'Pipeline finished.'
        }
    }
}
