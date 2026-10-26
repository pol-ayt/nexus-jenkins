pipeline {

    agent {

        kubernetes {

            cloud 'eks-dynamic-agents'

            namespace 'jenkins-agents'

            yaml '''

apiVersion: v1

kind: Pod

spec:

  containers:

    - name: maven

      image: maven:3.9.9-eclipse-temurin-17

      command: ["sleep"]

      args: ["99d"]

      tty: true

      resources:

        requests:

          cpu: "500m"

          memory: "1Gi"

        limits:

          memory: "2Gi"

'''

        }

    }

    stages {

        stage('Checkout') {

            steps {

                git branch: 'main', url: 'https://github.com/pol-ayt/nexus-jenkins.git'

            }

        }

        stage('Build') {

            steps {

                container('maven') {

                    sh 'mvn -B clean package'

                }

            }

        }

        stage('Verify Artifact') {

            steps {

                container('maven') {

                    sh 'ls -lh target/*.jar'

                }

            }

        }

        stage('Archive Artifact') {

            steps {

                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true

            }

        }

    }

}