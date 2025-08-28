pipeline {
  agent any
  tools {
    jdk   'jdk17'
    maven 'maven-3.9'
  }

 environment {
    // Nexus creds: creates $NEXUS_CREDS, $NEXUS_CREDS_USR, $NEXUS_CREDS_PSW
    NEXUS_CREDS = credentials('nexus-creds')

    // GAV + Nexus repos
    GROUP_ID       = 'com.example'
    ARTIFACT_ID    = 'hello-world'
    VERSION        = '1.0.0-SNAPSHOT'
    NEXUS_BASE     = 'https://nexus.clarusway.us'
  }

  stages {
    stage('Prep Java & Maven') {
      steps {
        sh '''
          echo "Using toolchain (from tools{}):"
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
          chmod 600 settings.xml
          # Optional: show sanitized preview (mask password)
          sed -e "s#<password>.*</password>#<password>***</password>#g" settings.xml | head -n 40
        '''
      }
    }

    stage('Build & Publish (deploy)') {
      steps {
        sh '''
          mvn -s settings.xml -DskipTests clean deploy
        '''
      }
    }

    stage('Fetch from Nexus (resolve SNAPSHOT via Maven)') {
      steps {
        sh '''
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
          java -jar app.jar
        '''
      }
    }
  }

  post {
    always {
      sh 'shred -u settings.xml 2>/dev/null || rm -f settings.xml' // we can delete the settings.xml to hide user sensetive data. shred -u = overwrite the file with random data before deleting.
      archiveArtifacts artifacts: 'app.jar, app.log', onlyIfSuccessful: false
    }
  }
}