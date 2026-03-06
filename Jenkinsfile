pipeline {
  agent { label 'maven-agent' }

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    // ---- Application Server details (change if needed) ----
    APP_SERVER_IP   = "10.2.1.186"
    APP_SERVER_USER = "ubuntu"
    APP_SERVER_DIR  = "/home/ubuntu/app"

    // ---- Security threshold (CVSS 0-10) ----
    // Pipeline fails if vulnerabilities >= this score are found
    FAIL_CVSS = "7"
  }

  stages {

    // 1) Checkout Code
    stage('Checkout Code') {
      steps {
        checkout scm
      }
    }

    // 2) Build & Test using Maven
    stage('Build & Test') {
      steps {
        sh 'mvn -v'
        sh 'mvn clean test'
      }
    }

    // 3) Security Scan Stage (OWASP Dependency-Check)
    stage('Security Scan') {
      steps {
        // Runs OWASP Dependency-Check using the Maven plugin
        // Fails build if any dependency vulnerability has CVSS >= FAIL_CVSS
        sh """
          mvn -DskipTests \
             org.owasp:dependency-check-maven:check \
             -Dformat=HTML \
             -DfailBuildOnCVSS=${FAIL_CVSS}
        """
      }
      post {
        always {
          // Archive the report if generated (path may vary)
          archiveArtifacts artifacts: '**/dependency-check-report.*', allowEmptyArchive: true
        }
      }
    }

    // 4) Package Stage
    stage('Package') {
      steps {
        sh 'mvn -DskipTests package'
      }
      post {
        success {
          // Archive your packaged artifact (adjust if WAR or different name)
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
      }
    }

    // 5) Deploy to Application Server (only for main branch)
    stage('Deploy to Application Server') {
      when {
        anyOf {
          branch 'main'
          branch 'master'   // include this because many repos still use master
        }
      }

      steps {
        // Uses your Jenkins credential ID created earlier: app-server-ssh
        sshagent(['app-server-ssh']) {
          sh """
            set -e

            echo "Creating app directory on server..."
            ssh -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} "mkdir -p ${APP_SERVER_DIR}"

            echo "Copying artifact to Application Server..."
            scp -o StrictHostKeyChecking=no target/*.jar ${APP_SERVER_USER}@${APP_SERVER_IP}:${APP_SERVER_DIR}/app.jar

            echo "Restarting application on Application Server..."
            ssh -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} '
              set -e
              # Stop old app (simple method: kill any process running app.jar)
              pkill -f "${APP_SERVER_DIR}/app.jar" || true
              nohup java -jar ${APP_SERVER_DIR}/app.jar > ${APP_SERVER_DIR}/app.log 2>&1 &
              sleep 2
              pgrep -f "${APP_SERVER_DIR}/app.jar" >/dev/null
              echo "App restarted successfully."
            '
          """
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline finished. Check archived artifacts and logs if needed."
    }
  }
}
