pipeline {
  agent any

  options { timestamps() }

  environment {
    // So "from app..." works when pytest runs
    PYTHONPATH = "${WORKSPACE}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Setup Python + deps') {
      steps {
        ansiColor('xterm') {
          sh '''
            set -eux
            python3 --version

            # fresh venv
            python3 -m venv venv
            . venv/bin/activate

            # Use venv's python to drive pip; do NOT upgrade system pip
            venv/bin/python -m pip --version
            venv/bin/python -m pip install --no-cache-dir -r requirements.txt
          '''
        }
      }
    }

    stage('Run Unit Tests') {
      steps {
        ansiColor('xterm') {
          sh '''
            set -eux
            . venv/bin/activate
            # PYTHONPATH already exported at pipeline level
            venv/bin/python -m pytest -q --maxfail=1 --disable-warnings
          '''
        }
      }
    }
  }

  post {
    success {
      withCredentials([
        string(credentialsId: 'webex-token',   variable: 'WEBEX_TOKEN'),
        string(credentialsId: 'webex-room-id', variable: 'WEBEX_ROOM_ID')
      ]) {
        ansiColor('xterm') {
          // Build JSON safely with a heredoc; shell expands env vars; Groovy does not.
          sh '''
            set -e
            cat <<EOF | curl -sS https://webexapis.com/v1/messages \
              -H "Authorization: Bearer $WEBEX_TOKEN" \
              -H "Content-Type: application/json" \
              -d @-
            {
              "roomId": "$WEBEX_ROOM_ID",
              "markdown": "**SUCCESS** :white_check_mark: Job *$JOB_NAME* #$BUILD_NUMBER passed tests."
            }
EOF
          '''
        }
      }
    }
    failure {
      withCredentials([
        string(credentialsId: 'webex-token',   variable: 'WEBEX_TOKEN'),
        string(credentialsId: 'webex-room-id', variable: 'WEBEX_ROOM_ID')
      ]) {
        ansiColor('xterm') {
          sh '''
            set -e
            cat <<EOF | curl -sS https://webexapis.com/v1/messages \
              -H "Authorization: Bearer $WEBEX_TOKEN" \
              -H "Content-Type: application/json" \
              -d @-
            {
              "roomId": "$WEBEX_ROOM_ID",
              "markdown": "**FAILURE** :x: Job *$JOB_NAME* #$BUILD_NUMBER failed. Check Jenkins console."
            }
EOF
          '''
        }
      }
    }
  }
}
