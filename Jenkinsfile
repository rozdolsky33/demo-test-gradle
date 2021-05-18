def templatePath = 'https://raw.githubusercontent.com/rozdolsky33/demo-test-gradle/pipeline/pipeline.json'
def templateName = 'demo-test-backend'
pipeline {
  agent {
    node {
      label 'gradle'
    }
  }
  options {
    timeout(time: 50, unit: 'MINUTES')
  }
  stages {
    stage('Preamble') {
        steps {
            script {
                openshift.withCluster() {
                    openshift.withProject() {
                        echo "Using project: ${openshift.project()}"
                    }
                }
            }
        }
    }
    stage('Cluster Cleanup') {
      steps {
        script {
            openshift.withCluster() {
                openshift.withProject() {
                  openshift.selector("all", [ template : templateName ]).delete()
                  if (openshift.selector("secrets", templateName).exists()) {
                    openshift.selector("secrets", templateName).delete()
                  }
                }
            }
        }
      }
    }
    stage('Clean'){
       steps {
          sh './gradlew --stacktrace clean'
       }
    }
    stage('Check'){
      steps {
        sh './gradlew --stacktrace check'
      }
    }
    stage('Test'){
      steps {
        sh './gradlew --stacktrace test'
        junit 'build/test-results/**/*.xml'
      }
    }
    stage('Create App') {
      steps {
        script {
            openshift.withCluster() {
                openshift.withProject() {
                  openshift.newApp(templatePath)
                }
            }
        }
      }
    }
    stage('Build') {
      steps {
          sh './gradlew --stacktrace build'
          archiveArtifacts artifacts: 'build/libs/**/*.jar', fingerprint: true
          echo "Running ${env.BUILD_ID}"
      }
      steps {
        script {
            openshift.withCluster() {
                openshift.withProject() {
                  def builds = openshift.selector("bc", templateName).related('builds')
                  timeout(10) {
                    builds.untilEach(1) {
                      return (it.object().status.phase == "Complete")
                    }
                  }
                }
            }
        }
      }
    }
    stage('Publish Artifact to Nexus'){
      steps {
        nexusArtifactUploader artifacts: [
                            [artifactId: 'demo-test-backend',
                             classifier: '',
                             file: 'build/libs/workspace-0.0.1.jar',
                             type: 'jar']],
                             credentialsId: 'nexuslogin',
                             groupId: 'DEV',
                             nexusUrl: 'nexus-nexus.bnsf-dev-03-e648741016b5b16f9b585588dcd0ed80-0000.us-south.containers.appdomain.cloud',
                             nexusVersion: 'nexus3',
                             protocol: 'http',
                             repository: 'spec-test-release',
                             version: 'Version.1.0.${BUILD_ID}'
       }
    }
    stage('deploy') {
      steps {
        script {
            openshift.withCluster() {
                openshift.withProject() {
                  def rm = openshift.selector("dc", templateName).rollout()
                  timeout(5) {
                    openshift.selector("dc", templateName).related('pods').untilEach(1) {
                      return (it.object().status.phase == "Running")
                    }
                  }
                }
            }
        }
      }
    }
    stage('tag') {
      steps {
        script {
            openshift.withCluster() {
                openshift.withProject() {
                  openshift.tag("${templateName}:latest", "${templateName}-staging:latest")
                }
            }
        }
      }
    }
  }
}
