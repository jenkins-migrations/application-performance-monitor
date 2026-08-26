<![CDATA[
// Source: https://github.com/jenkinsci/pipeline-examples (MIT License)
// Test case: External workspace management  
def extWorkspace = exwsAllocate 'diskpool1'

node('linux') {
    exws(extWorkspace) {
        stage('Checkout') {
            checkout scm
        }
        
        stage('Build') {
            sh 'mvn clean install -DskipTests'
        }
        
        stage('Package') {
            sh 'mvn package -DskipTests'
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
    }
}

node('test') {
    exws(extWorkspace) {
        try {
            stage('Unit Tests') {
                sh 'mvn test'
                publishTestResults testResultsPattern: 'target/surefire-reports/*.xml'
            }
            
            stage('Integration Tests') {
                sh 'mvn failsafe:integration-test'
                publishTestResults testResultsPattern: 'target/failsafe-reports/*.xml'
            }
        } catch (e) {
            currentBuild.result = 'FAILURE'
            throw e
        } finally {
            cleanWs cleanWhenFailure: false
        }
    }
}
    ]]>