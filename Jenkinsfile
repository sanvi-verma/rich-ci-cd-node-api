pipeline {
    agent any

    environment {
        SONAR_SCANNER_HOME = tool 'SonarQubeScanner' // Add this tool in Jenkins global tools config
        SONAR_PROJECT_KEY = 'node-api'
        SONAR_ORG = 'your-org' // if using SonarCloud, else remove
        WEBHOOK_URL = 'https://your-api-endpoint.com/receive' // Replace with your actual API URL
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/sanvi-verma/rich-ci-cd-node-api.git', branch: 'main'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Lint') {
            steps {
                sh 'npx eslint . || true' // use || true if you want to not fail on warnings
            }
        }

        stage('Test') {
            steps {
                sh 'npx mocha'
            }
        }

        stage('Coverage') {
            steps {
                sh 'npx nyc --reporter=lcov --reporter=text mocha'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SONAR_TOKEN = credentials('sonar-token') // Add your token in Jenkins credentials
            }
            steps {
                withSonarQubeEnv('My SonarQube Server') {
                    sh '''
                       npx sonar-scanner \
  -Dsonar.projectKey=node-api \
  -Dsonar.sources=. \
  -Dsonar.exclusions=test/** \
  -Dsonar.tests=test \
  -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
  -Dsonar.host.url=http://host.docker.internal:9000 \
  -Dsonar.token=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh 'nohup npm start &'
            }
        }
    }

    post {
        always {
            script {
                withCredentials([
                    string(credentialsId: 'jenkins-username', variable: 'JENKINS_USERNAME'),
                    string(credentialsId: 'api-token', variable: 'API_TOKEN'),
                    string(credentialsId: 'secret-key', variable: 'SECRET_KEY'),
                    string(credentialsId: 'iv-key', variable: 'IV_KEY')
                ]) {
                    def getRawJson = { url ->
                        sh(script: "curl -s -u '$JENKINS_USERNAME:$API_TOKEN' '${url}'", returnStdout: true).trim()
                    }

                    def buildData = getRawJson("${env.JENKINS_URL}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}/api/json")
                    def stageDescribe = getRawJson("${env.JENKINS_URL}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}/wfapi/describe")

                    writeFile file: 'stageDescribe.json', text: stageDescribe
                    def parsedDescribe = readJSON file: 'stageDescribe.json'

                    def nodeStageDataStr = parsedDescribe.stages.collect { stage ->
                        def nodeId = stage.id
                        def nodeData = getRawJson("${env.JENKINS_URL}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}/execution/node/${nodeId}/wfapi/describe")
                        return """{"nodeId":${groovy.json.JsonOutput.toJson(nodeId)},"data":${nodeData}}"""
                    }.join(',')

                    def payloadStr = """{"build_data": ${buildData}, "node_stage_data": [${nodeStageDataStr}]}"""
                    writeFile file: 'payload.json', text: payloadStr

                    def checksum = sh(script: "sha256sum payload.json | awk '{print \$1}'", returnStdout: true).trim()

                    def timestamp = System.currentTimeMillis().toString()
                    def encryptedTimestamp = sh(script: """
        echo -n '${timestamp}' | openssl enc -aes-256-cbc -base64 \\
        -K ${SECRET_KEY} \\
        -iv ${IV_KEY}
    """, returnStdout: true).trim()

                    sh """
                        curl -X POST '${WEBHOOK_URL}' \\
                        -H "Content-Type: application/json" \\
                          -H "X-Encrypted-Timestamp: ${encryptedTimestamp}" \\
                        -H "X-Checksum: ${checksum}" \\
                        --data-binary @payload.json
                    """
                }
            }
        }
    }
}
