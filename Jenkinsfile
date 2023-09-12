def E2E = 'False'
def TAG = "1.0.0"
def TAGTEL = "1.0.0"
def JARTM = ""
def JARSIM = ""

pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
    }

    environment {
        MVN="mvn -s settings.xml"
    }

    stages {
        stage('Git-checkout'){
            steps {
                deleteDir()
                checkout scm
            }
        }

        stage('Check-last-commit'){
            when {
                branch 'feature/*'
            }

            agent {
                docker {
                    image 'maven:3.6.3-jdk-8'
                    args '--network jenkins_jenkins_network'
                }
            }

            steps {
                script {
                    def lastCommitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true)
                    if (lastCommitMessage.contains("#e2e")) {
                        E2E = 'True'
                    } else {
                        sh "${MVN} package"
                    }
                }
            }
        }

        stage ('Get-version'){
            when {
                branch 'release/*'
            }

            steps {
                script {
                    def version = env.BRANCH_NAME.split('/')[1]
                    echo "${version}"
                    def tag_c = "0"
                    def tag_tel = "0"
                    sshagent(credentials: ['GitlabSSHprivateKey']){
                        sh "git ls-remote --tags origin | grep 1.0 | wc -l"
                        tag_tel = sh(script: "git ls-remote --tags git@gitlab.com:amitlubin/exam2_telemetry.git | grep ${version} | wc -l", returnStdout: true)
                        tag_c = sh(script: "git ls-remote --tags origin | grep ${version} | wc -l", returnStdout: true)
                    }

                    def new_tag = (tag_tel.toInteger() - 1).toString()
                    echo "${new_tag}"
                    def tag_untrimmed = "${version}.${tag_c}"
                    TAG = tag_untrimmed.trim()
                    def tag_tel_untrimmed = "${version}.${new_tag}"
                    TAGTEL = tag_tel_untrimmed.trim()
                    echo "${TAGTEL}"
                }
            }
        }

        stage('Put-version-and-maven-build'){
            when {
                branch 'release/*'
            }

            agent {
                docker {
                    image 'maven:3.6.3-jdk-8'
                    args '--network jenkins_jenkins_network'
                }
            }

            steps {
                sh "${MVN} versions:set -DnewVersion=${TAG}"
                sh "${MVN} verify"
            }
        }

        stage('Maven-build'){
            when {
                anyOf {
                    branch 'main'
                    expression {
                        return (env.BRANCH_NAME =~ /^feature\/.*/ && E2E == 'True')
                    }
                }
            }

            agent {
                docker {
                    image 'maven:3.6.3-jdk-8'
                    args '--network jenkins_jenkins_network'
                }
            }

            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        echo "Skipped!"
                        sh "${MVN} verify -DskipTests"
                    } else {
                        echo "Tested!"
                        sh "${MVN} verify"
                    }
                }
            }
        }

        stage('Curl-artifactory-and-E2E'){
            when {
                anyOf {
                    branch 'main'
                    expression {
                        return (env.BRANCH_NAME =~ /^feature\/.*/ && E2E == 'True')
                    }
                }
            }

            agent {
                docker {
                    // image 'openjdk:8-jre-alpine3.9'
                    image 'maven:3.6.3-jdk-8'
                    args '--network jenkins_jenkins_network'
                }
            }

            steps {
                script {
                    def telemetry = sh(script: "curl -u admin:Al12341234 -X GET 'http://artifactory:8082/artifactory/api/storage/libs-snapshot-local/com/lidar/telemetry/99-SNAPSHOT/'", returnStdout: true)
                    def simulator = sh(script: "curl -u admin:Al12341234 -X GET 'http://artifactory:8082/artifactory/api/storage/libs-snapshot-local/com/lidar/simulator/99-SNAPSHOT/'", returnStdout: true)

                    def jsonSlurper = new groovy.json.JsonSlurper()
                    def parsedTelemetry= jsonSlurper.parseText(telemetry)
                    def parsedSimulator = jsonSlurper.parseText(simulator)

                    // Extract the JAR file URI
                    def jarTelemetry = parsedTelemetry.children.find { it.uri.endsWith(".jar") }?.uri
                    def jarSimulator = parsedSimulator.children.find { it.uri.endsWith(".jar") }?.uri

                    echo "${jarTelemetry}"
                    echo "${jarSimulator}"

                    JARTM = jarTelemetry
                    JARSIM = jarSimulator
                }
            }
        }

        stage('not-release-test'){
            when {
                anyOf {
                    branch 'main'
                    expression {
                        return (env.BRANCH_NAME =~ /^feature\/.*/ && E2E == 'True')
                    }
                }
            }

            agent {
                docker {
                    image 'maven:3.6.3-jdk-8'
                    args '--network jenkins_jenkins_network'
                }
            }

            steps {
                sh "curl -u admin:Al12341234 -O 'http://artifactory:8082/artifactory/libs-snapshot-local/com/lidar/telemetry/99-SNAPSHOT${JARTM}'"
                sh "curl -u admin:Al12341234 -O 'http://artifactory:8082/artifactory/libs-snapshot-local/com/lidar/simulator/99-SNAPSHOT${JARSIM}'"
                sh "ls -l"
                sh "ls target"
                sh "java -cp .${JARSIM}:.${JARTM}:target/analytics-99-SNAPSHOT.jar com.lidar.simulation.Simulator"
            }
        }

        stage("Release-artifactory"){
            when {
                branch 'release/*'
            }

            agent {
                docker {
                    image 'maven:3.6.3-jdk-8'
                    args '--network jenkins_jenkins_network'
                }
            }

            steps {
                script {
                    echo "${TAGTEL}"
                    def url = "http://artifactory:8082/artifactory/api/storage/libs-release-local/com/lidar/telemetry/${TAGTEL}"
                    echo "${url}"

                    def telemetry = sh(script: "curl -u admin:Al12341234 -X GET ${url}", returnStdout: true)
                    def simulator = sh(script: "curl -u admin:Al12341234 -X GET 'http://artifactory:8082/artifactory/api/storage/libs-snapshot-local/com/lidar/simulator/99-SNAPSHOT/'", returnStdout: true)

                    echo "Passed"
                    
                    def jsonSlurper = new groovy.json.JsonSlurper()
                    def parsedTelemetry= jsonSlurper.parseText(telemetry)
                    def parsedSimulator = jsonSlurper.parseText(simulator)

                    // Extract the JAR file URI
                    def jarTelemetry = parsedTelemetry.children.find { it.uri.endsWith(".jar") }?.uri
                    def jarSimulator = parsedSimulator.children.find { it.uri.endsWith(".jar") }?.uri

                    echo "${jarTelemetry}"
                    echo "${jarSimulator}"

                    JARTM = jarTelemetry
                    JARSIM = jarSimulator
                }
            }
        }

        stage('Test'){
            when {
                branch 'release/*'
            }

            agent {
                docker {
                    image 'maven:3.6.3-jdk-8'
                    args '--network jenkins_jenkins_network'
                }
            }

            steps {
                
                sh "curl -u admin:Al12341234 -O http://artifactory:8082/artifactory/libs-release-local/com/lidar/telemetry/${TAGTEL}${JARTM}"
                sh "curl -u admin:Al12341234 -O 'http://artifactory:8082/artifactory/libs-snapshot-local/com/lidar/simulator/99-SNAPSHOT${JARSIM}'"
                sh "ls -l"
                sh "ls target"
                sh "java -cp .${JARSIM}:.${JARTM}:target/analytics-${TAG}.jar com.lidar.simulation.Simulator"

                stash(name: 'jar', includes: 'target/*.jar')
            }
        }

        stage('Maven-deploy'){
            when {
                anyOf {
                    branch 'main'
                    branch 'release/*'
                    expression {
                        return (env.BRANCH_NAME =~ /^feature\/.*/ && E2E == 'True')
                    }
                }
            }

            agent {
                docker {
                    image 'maven:3.6.3-jdk-8'
                    args '--network jenkins_jenkins_network'
                }
            }

            steps {
                script {
                    if (env.BRANCH_NAME =~ /^release\/.*/){
                        sh "${MVN} versions:set -DnewVersion=${TAG}"
                        echo "Versioned!"
                    }

                    sh "${MVN} deploy -DskipTests"
                }
            }
        }

        stage('Git-tag'){
            when {
                branch 'release/*'
            }

            steps {
                unstash(name: 'jar')

                sshagent(credentials: ['GitlabSSHprivateKey']){
                    sh "git tag v${TAG}"
                    sh "git push origin v${TAG}"                    
                }
            }

        }


    }

    post {
        always {
            cleanWs()
        }
    }
}