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

    stage('Fetch from Nexus (resolve SNAPSHOT via Maven)') {
        steps {
            // Uses mirror in settings.xml -> maven-public (which includes snapshots)
            sh '''
            export PATH="$JDK_HOME/bin:$MAVEN_HOME/bin:$PATH"
            # This resolves the latest SNAPSHOT and copies the JAR to the current directory
            mvn -s settings.xml -q org.apache.maven.plugins:maven-dependency-plugin:3.6.1:copy \
                -Dartifact="$GROUP_ID:$ARTIFACT_ID:$VERSION:jar" \
                -DoutputDirectory=. \
                -DoverWrite=true

            ls -lh *.jar
            # Rename to app.jar for the next stage
            mv ${ARTIFACT_ID}-*.jar app.jar
            ls -lh app.jar
            '''
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