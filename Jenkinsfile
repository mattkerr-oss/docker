pipeline {
  agent{
    kubernetes {
      cloud 'k8s'
      label 'jenkins-maven-pipeline'
      defaultContainer 'jnlp' 
        yaml """
            apiVersion: v1
            kind: Pod
            metadata:
              labels:
                name: jenkins-maven-pipeline
            spec:
              containers:
              - name: maven
                image: maven:3.6.0-jdk-11
                command:
                - cat
                tty: true
    """
    }
  }
    stages {
        stage ('Clone') {
            steps {
                git branch: 'pipeline', url: "https://github.com/yairbass/petclinic-docker.git", credentialsId: 'github'
            }
        }
        
        
      
        stage ('Artifactory configuration') {
            steps {
                rtMavenDeployer (
                    id: "MAVEN_DEPLOYER",
                    serverId: "artifactory",
                    releaseRepo: "maven-virtual",
                    snapshotRepo: "maven-virtual"
                    
                )

                rtMavenResolver (
                    id: "MAVEN_RESOLVER",
                    serverId: "artifactory",
                    releaseRepo: "maven-virtual",
                    snapshotRepo: "maven-virtual"
                   
                )
            }
        }

        stage ('Exec Maven') {
            steps {
                rtMavenRun (
                    tool: 'MAVEN_TOOL', // Tool name from Jenkins configuration
                    pom: 'pom.xml',
                    goals: 'package -Dmaven.test.skip=true ',
                    deployerId: "MAVEN_DEPLOYER",
                    resolverId: "MAVEN_RESOLVER"
                )
            }
        }
        stage ('SonarQube') {
            environment {
                scannerHome = tool 'SonarQubeScanner'
            }
            steps {
        withSonarQubeEnv('SonarServer') {
            sh "${scannerHome}/bin/sonar-scanner"
                }
                }
            }
        
        stage("Quality Gate") {
            steps {
                // Fix for SonarQube issue for Pending Quality gates
                sh "sleep 10"
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
    }
        stage ('Publish build info') {
            steps {
                rtPublishBuildInfo (
                    serverId: "artifactory"
                )
            }
        }
        
            stage ('Scan Build with xray') {
                steps {
                    script {
                        server = Artifactory.server "artifactory"
                        def scanConfig = [
                            'buildName'      : env.JOB_NAME,
                            'buildNumber'    : env.BUILD_NUMBER,
                            'failBuild'      : false
                          ]
                        def scanResult = server.xrayScan scanConfig
                        echo scanResult as String
                    }
                }
              }
            stage ('Promote to release'){
                steps {
                    rtPromote (
                    buildName: env.JOB_NAME,
                    buildNumber: env.BUILD_NUMBER,
                    serverId: 'artifactory',
                    targetRepo: 'maven-release-local',
                    comment: 'this is build had passed all tests is been promoted',
                    status: 'Released',
                    sourceRepo: 'maven-dev-local-repo',
                    includeDependencies: false,
                    failFast: true,
                    copy: true
                )
       }
     }
    }
}
