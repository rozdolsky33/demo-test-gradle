def templatePath = 'https://raw.githubusercontent.com/rozdolsky33/demo-test-gradle/pipeline-v3/pipeline.yaml'
def templateName = 'demo-test-backend'
def devTag  = 'Version.1.0.${BUILD_ID}'
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
        stage('Build Artifact') {
          steps {
              sh './gradlew --stacktrace build'
              archiveArtifacts artifacts: 'build/libs/**/*.jar', fingerprint: true
              echo "Running ${env.BUILD_ID}"
          }
        }
        stage('Publish Artifact To Nexus'){
          steps {
            nexusArtifactUploader artifacts: [
                                [artifactId: 'demo-test-backend',
                                 classifier: '',
                                 file: 'build/libs/demo-test-gradle-0.0.1-SNAPSHOT.jar',
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
        stage('Preamble Dev') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('bnsf-dev') {
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
                    openshift.withProject('bnsf-dev') {
                      openshift.selector("all", [ template : templateName ]).delete()
                      if (openshift.selector("secrets", templateName).exists()) {
                        openshift.selector("secrets", templateName).delete()
                      }
                    }
                }
            }
          }
        }
        stage('Create App Template Dev') {
          steps {
            script {
                openshift.withCluster() {
                    openshift.withProject('bnsf-dev') {
                      openshift.newApp(templatePath)
                    }
                }
            }
          }
        }
        stage('Build Dev Deployment') {
          steps {
            script {
                openshift.withCluster() {
                    openshift.withProject('bnsf-dev') {
                      echo " BUILDING DEV"
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
        stage('Tag-dev') {
          steps {
            script {
                openshift.withCluster() {
                    openshift.withProject('bnsf-dev') {
                      echo "TAG DEV"
                      openshift.tag("${templateName}", "${templateName}-dev:latest")

                    }
                }
            }
          }
        }
        stage('Deploy To Dev') {
          steps {
            script {
                openshift.withCluster() {
                    openshift.withProject('bnsf-dev') {
                    echo "DEPLOY TO DEV"
                      def rm = openshift.selector("dc", "${templateName}-dev").rollout()
                      timeout(5) {
                        openshift.selector("dc", "${templateName}-dev").related('pods').untilEach(1) {
                          return (it.object().status.phase == "Running")
                        }
                      }
                   }
                }
            }
          }
        }
        stage('Tag staging') {
          steps {
            script {
                openshift.withCluster() {
                    openshift.withProject('bnsf-dev') {
                      openshift.tag("${templateName}-dev:latest", "${templateName}-staging:latest")
                    }
                }
              }
          }
        }
        stage('Preamble Stage') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('bnsf-stage') {
                            echo "Using project: ${openshift.project()}"
                        }
                    }
                }
            }
        }
        stage('Create App Template Stage') {
          steps {
            script {
                openshift.withCluster() {
                    openshift.withProject('bnsf-stage') {
                      echo "Building From Template"
                      openshift.newApp(templatePath)
                      echo "Created template with new-app"
                    }
                }
            }
          }
        }
        stage('Build Stage Deployment') {
          steps {
            script {
                openshift.withCluster() {
                    openshift.withProject('bnsf-stage') {
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
        stage('Deploy To Stage') {
          steps {
            script {
              openshift.withCluster() {
                openshift.withProject('bnsf-stage') {
                def rm = openshift.selector("dc", "${templateName}-staging").rollout()
                  timeout(5) {
                    openshift.selector("dc", "${templateName}-staging").related('pods').untilEach(1) {
                          return (it.object().status.phase == "Running")
                      } //selector
                    } // timeout
                  } // withProject
               } // withCluster
            }//script
          }//steps
        }//stage
     } // stages
 }//pipeline
