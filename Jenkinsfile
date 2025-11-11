pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }
  environment {
    WEBEX_ROOM_ID = credentials('webex-room-id')
    WEBEX_TOKEN   = credentials('webex-token')
  }
  triggers { pollSCM('@daily') } // real trigger will be GitHub webhook
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Setup Python + deps') {
      steps {
        sh '''
          set -eux
          python3 --version || true
          if ! command -v python3 >/dev/null 2>&1; then
            echo "Install Python inside Jenkins container once (later step)."
            exit 1
          fi
          python3 -m venv venv
          . venv/bin/activate
          pip install -U pip
          pip install -r requirements.txt
        '''
      }
    }
    stage('Run Unit Tests') {
      steps {
        sh '''
          set -eux
          . venv/bin/activate
          pytest -q --maxfail=1 --disable-warnings
        '''
      }
    }
  }
  post {
    success {
      sh """
        curl -sS https://webexapis.com/v1/messages \
          -H "Authorization: Bearer ${WEBEX_TOKEN}" \
          -H "Content-Type: application/json" \
          -d '{ "roomId": "${WEBEX_ROOM_ID}", "markdown": "**SUCCESS** :white_check_mark: Job *${JOB_NAME}* #${BUILD_NUMBER} on *${BRANCH_NAME}* passed tests." }' >/dev/null
      """
    }
    failure {
      sh """
        curl -sS https://webexapis.com/v1/messages \
          -H "Authorization: Bearer ${WEBEX_TOKEN}" \
          -H "Content-Type: application/json" \
          -d '{ "roomId": "${WEBEX_ROOM_ID}", "markdown": "**FAILURE** :x: Job *${JOB_NAME}* #${BUILD_NUMBER} failed. Check Jenkins console." }' >/dev/null
      """
    }
  }
}
