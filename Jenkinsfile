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
    withSonarQubeEnv('Default SonarQube Server'){
      sh 'mvn clean verify sonar:sonar'
    }
    //sh 'mvn clean verify sonar:sonar -Dsonar.projectName=example-project -Dsonar.projectKey=example-project -Dsonar.projectVersion=$BUILD_NUMBER';
  }
  stage("Quality Gate"){
    timeout(time: 1, unit: 'HOURS') {
      def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
      if (qg.status != 'OK') {
        error "Pipeline aborted due to quality gate failure: ${qg.status}"
      }
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
  stash includes:
    'target/hello-0.0.1.war,src/pt/Hello_World_Test_Plan.jmx', name:'binary'
}
node('docker_pt')
{
  stage('Start Tomcat')
  {
    sh '''cd /home/jenkins/tomcat/bin
    ./startup.sh''';
  }
  stage('Deploy')
  {
    unstash 'binary'
    sh 'cp target/hello-0.0.1.war /home/jenkins/tomcat/webapps/';
  }
  stage('Performance Testing')
  {
    sh '''cd /opt/jmeter/bin/
    ./jmeter.sh -n -t $WORKSPACE/src/pt/Hello_World_Test_Plan.jmx -l $WORKSPACE/test_report.jtl''';
    step([$class: 'ArtifactArchiver', artifacts: '**/*.jtl'])
  }
  stage('Promote build in Artifactory')
  {
    withCredentials([usernameColonPassword(credentialsId:'artifactory-account', variable:'credentials')]){
      sh 'curl -u${credentials} -X PUT "http://14.42.235.248:49153/artifactory/api/storage/example-project/${BUILD_NUMBER}/hello-0.0.1.war properties=Performance-Tested=YES"';
    }
  }
}
node('production'){
  stage('Deploy and Prod'){
    def server = Artifactory.server 'Default Artifactory Server'
    def downloadSpec= """{
    "files" : [
      {
        "pattern":"example-project/$BUILD_NUMBER/*.zip",
        "target":"/home/ubuntu/tomcat/webapps/",
        "props":"Performance-Tested=Yes;Intergration-Tested=Yes"
      }
    ]
  }"""
    server.download(downloadSpec)
  }
}
