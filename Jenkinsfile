node('docker')
{
  stage('Poll')
  {
    checkout scm
  }
  stage('Build &  Unit test')
  {
    sh 'mvn clean verify -DskipITs=true';
    junit '**/target/surefire-reports/TEST-*.xml'
    archive 'target/*.jar'
  }
  stage('Static Code Analysis')
  {
    sh 'mvn clean verify sonar:sonar -Dsonar.projectName=example-project -Dsonar.projectKey=example-project -Dsonar.projectVersion=$BUILD_NUMBER';
  }
  stage("Quality Gate"){
    timeout(time: 1, unit: 'HOURS') {
      def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
      if (qg.status != 'OK') {
        error "Pipeline aborted due to quality gate failure: ${qg.status}
      }
  }
  stage('Intergration Test')
  {
    sh 'mvn clean verify -Dsurefire.skip=true';
    junit '**/target/failsafe-reports/TEST-*.xml'
    archive 'target/*.jar'
  }
  stage('Publish')
  {
    def server = Artifactory.server 'Default Artifactory Server'
    def uploadSpec = """{
      "files":[
      {
        "pattern":"target/hello-0.0.1.war",
        "target":"example-project/${BUILD_NUMBER}/",
        "props":"Intergration-Tested=Yes;Performance-Tested=No"
      }]
    }"""
    server.upload(uploadSpec)
  }
}
