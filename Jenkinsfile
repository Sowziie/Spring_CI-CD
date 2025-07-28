pipeline {
  agent any

  tools {
    jdk   'jdk17'
    maven 'maven3'
  }

  environment {
    DOCKER_CRED = 'dockerhub'
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
    script {
      def mvnHome = tool 'maven3'
      withSonarQubeEnv('MySonar') {
        sh """
          ${mvnHome}/bin/mvn clean verify sonar:sonar \
          -Dsonar.projectKey=Sonar \
          -Dsonar.projectName='Sonar' \
          -Dsonar.token=$SONAR_AUTH_TOKEN
        """
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