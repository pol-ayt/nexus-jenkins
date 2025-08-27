pipeline {
  agent any

  environment {
    JDK_HOME   = tool 'jdk17'
    MAVEN_HOME = tool 'maven-3.9'

    GROUP_ID    = 'com.example'
    ARTIFACT_ID = 'hello-world'
    VERSION     = '1.0.0-SNAPSHOT'

    NEXUS_BASE      = 'https://nexus.clarusway.us'
    RELEASES_REPO   = 'maven-releases'
    SNAPSHOTS_REPO  = 'maven-snapshots'
  }

  stages {
    stage('Prep Java & Maven') {
      steps {
        sh '''
          export PATH="$JDK_HOME/bin:$MAVEN_HOME/bin:$PATH"
          echo "Using:"
          java -version
          mvn -v
        '''
      }
    }

    stage('Write Maven settings.xml') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NUSER', passwordVariable: 'NPASS')]) {
          writeFile file: 'settings.xml', text: """
<settings>
  <servers>
    <server>
      <id>nexus-releases</id>
      <username>${NUSER}</username>
      <password>${NPASS}</password>
    </server>
    <server>
      <id>nexus-snapshots</id>
      <username>${NUSER}</username>
      <password>${NPASS}</password>
    </server>
  </servers>
  <mirrors>
    <mirror>
      <id>nexus-public</id>
      <mirrorOf>*</mirrorOf>
      <url>${NEXUS_BASE}/repository/maven-public/</url>
    </mirror>
  </mirrors>
</settings>
"""
        }
      }
    }

    stage('Build & Publish (deploy)') {
      steps {
        sh '''
          export PATH="$JDK_HOME/bin:$MAVEN_HOME/bin:$PATH"
          mvn -s settings.xml -DskipTests clean deploy
        '''
      }
    }

    stage('Fetch from Nexus') {
      steps {
        script {
          def repoName = VERSION.endsWith('-SNAPSHOT') ? SNAPSHOTS_REPO : RELEASES_REPO
          def groupPath = GROUP_ID.replace('.', '/')
          def jarName = "${ARTIFACT_ID}-${VERSION}.jar"
          def downloadUrl = "${NEXUS_BASE}/repository/${repoName}/${groupPath}/${ARTIFACT_ID}/${VERSION}/${jarName}"
          echo "Downloading: ${downloadUrl}"
          withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NUSER', passwordVariable: 'NPASS')]) {
            sh """
              curl -fSL -u "${NUSER}:${NPASS}" -o app.jar "${downloadUrl}"
              ls -lh app.jar
            """
          }
        }
      }
    }

    stage('Deploy (run the jar)') {
      steps {
        sh '''
          export PATH="$JDK_HOME/bin:$PATH"
          nohup java -jar app.jar > app.log 2>&1 &
          sleep 2
          pgrep -fl 'java.*app.jar' || (echo "App failed to start" && exit 1)
          echo "Last 20 lines of app.log:"
          tail -n 20 app.log || true
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'app.jar, app.log', onlyIfSuccessful: false
    }
  }
}