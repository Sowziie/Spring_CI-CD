pipeline {
  agent any

  tools {
    jdk   'jdk17'
    maven 'maven3'
  }

  environment {
    DOCKER_CRED = 'dockerhub'
    // SonarQube token référencé en Credential “Sonar”
    SONAR_TOKEN = credentials('Sonar')
    NEXUS_URL   = 'http://52.90.58.95/repository/maven-snapshots/'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Test') {
      steps { sh 'mvn clean package -B' }
    }

    stage('SonarQube Analysis') {
      steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
          // Injecte automatiquement SONAR_HOST_URL et SONAR_AUTH_TOKEN
          withSonarQubeEnv('MySonar') {
            // On passe uniquement le projectKey
            sh 'mvn sonar:sonar -Dsonar.projectKey=Sonar'
          }
        }
      }
    }

    stage('Docker Login, Build & Push') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: DOCKER_CRED,
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
            docker build -t $DOCKER_USER/demoapp:${GIT_COMMIT} .
            docker push $DOCKER_USER/demoapp:${GIT_COMMIT}
            docker tag $DOCKER_USER/demoapp:${GIT_COMMIT} $DOCKER_USER/demoapp:latest
            docker push $DOCKER_USER/demoapp:latest
          '''
        }
      }
    }

    stage('Deploy to Nexus') {
      steps {
        echo '📦 Déploiement du JAR vers Nexus (maven-snapshots)'
        withCredentials([usernamePassword(
          credentialsId: 'nexus-credentials',
          usernameVariable: 'NEXUS_USER',
          passwordVariable: 'NEXUS_PASS'
        )]) {
          sh '''cat > settings.xml <<EOF
<settings>
  <servers>
    <server>
      <id>nexus</id>
      <username>$NEXUS_USER</username>
      <password>$NEXUS_PASS</password>
    </server>
  </servers>
</settings>
EOF'''
          sh 'mvn deploy -B -s settings.xml -DaltDeploymentRepository=nexus::http://52.90.58.95:8081/repository/maven-snapshots/'
        }
      }
    }
  }

  post {
    success  { echo '✅ Pipeline terminé avec succès' }
    unstable { echo '⚠️ Pipeline instable (vérifier les logs)' }
    failure  { echo '❌ Pipeline échoué' }
  }
}
