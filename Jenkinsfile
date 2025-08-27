pipeline {
  agent any

  environment {
    // Tools (Manage Jenkins → Tools)
    JDK_HOME   = tool 'jdk17'
    MAVEN_HOME = tool 'maven-3.9'

    // Nexus creds: creates $NEXUS_CREDS, $NEXUS_CREDS_USR, $NEXUS_CREDS_PSW
    NEXUS_CREDS = credentials('nexus-creds')

    // GAV + Nexus repos
    GROUP_ID       = 'com.example'
    ARTIFACT_ID    = 'hello-world'
    VERSION        = '1.0.0-SNAPSHOT'
    NEXUS_BASE     = 'https://nexus.clarusway.us'
    RELEASES_REPO  = 'maven-releases'
    SNAPSHOTS_REPO = 'maven-snapshots'
  }

  stages {
    stage('Prep Java & Maven') {
      steps {
        sh '''
          export PATH="$JDK_HOME/bin:$MAVEN_HOME/bin:$PATH"
          echo "Using toolchain:"
          java -version
          mvn -v
        '''
      }
    }

    stage('Write Maven settings.xml (safe)') {
      steps {
        // Use heredoc so $NEXUS_CREDS_USR / $NEXUS_CREDS_PSW expand in the shell
        sh '''
          cat > settings.xml <<XML
<settings>
  <servers>
    <server>
      <id>nexus-releases</id>
      <username>$NEXUS_CREDS_USR</username>
      <password>$NEXUS_CREDS_PSW</password>
    </server>
    <server>
      <id>nexus-snapshots</id>
      <username>$NEXUS_CREDS_USR</username>
      <password>$NEXUS_CREDS_PSW</password>
    </server>
  </servers>
  <mirrors>
    <mirror>
      <id>nexus-public</id>
      <mirrorOf>*</mirrorOf>
      <url>$NEXUS_BASE/repository/maven-public/</url>
    </mirror>
  </mirrors>
</settings>
XML
          # Optional: show sanitized preview (mask password)
          sed -e "s#<password>.*</password>#<password>***</password>#g" settings.xml | head -n 40
        '''
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
        // maven-dependency-plugin resolves timestamped SNAPSHOTs automatically
        sh '''
          export PATH="$JDK_HOME/bin:$MAVEN_HOME/bin:$PATH"
          mvn -s settings.xml -q org.apache.maven.plugins:maven-dependency-plugin:3.6.1:copy \
             -Dartifact="$GROUP_ID:$ARTIFACT_ID:$VERSION:jar" \
             -DoutputDirectory=. \
             -DoverWrite=true

          # Rename the downloaded jar for the next stage
          mv ${ARTIFACT_ID}-*.jar app.jar
          ls -lh app.jar
        '''
      }
    }

    stage('Run app (simple)') {
      steps {
        sh '''
          export PATH="$JDK_HOME/bin:$PATH"
          java -jar app.jar
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
