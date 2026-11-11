pipeline {
    agent any

    options {
        timestamps()
    }

    triggers {
        // Fires when GitHub's webhook hits https://<ngrok-url>/github-webhook/
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Set up Python') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Unit tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    pytest -v --junitxml=test-results.xml
                '''
            }
            post {
                always {
                    junit 'test-results.xml'
                }
            }
        }
    }

    post {
        success {
            script { notifyWebex("SUCCESS") }
        }
        failure {
            script { notifyWebex("FAILURE") }
        }
        unstable {
            script { notifyWebex("UNSTABLE") }
        }
    }
}

def notifyWebex(String status) {
    def icon = (status == 'SUCCESS') ? '✅' : '❌'
    def text = "${icon} **${env.JOB_NAME}** build #${env.BUILD_NUMBER}: **${status}**\n\n[View console output](${env.BUILD_URL}console)"

    withCredentials([
        string(credentialsId: 'webex-bot-token', variable: 'WEBEX_TOKEN'),
        string(credentialsId: 'webex-room-id', variable: 'WEBEX_ROOM_ID')
    ]) {
        // Build the JSON body with Groovy so quotes/newlines in the message are escaped correctly
        def payload = groovy.json.JsonOutput.toJson([roomId: env.WEBEX_ROOM_ID, markdown: text])
        writeFile file: 'webex-payload.json', text: payload
        sh '''
            curl -sS -X POST https://webexapis.com/v1/messages \
                -H "Authorization: Bearer $WEBEX_TOKEN" \
                -H "Content-Type: application/json" \
                -d @webex-payload.json
        '''
    }
}
