pipeline {
  agent any

  options { timestamps() }

  environment {
    // make python see your repo root so "from app..." works
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
            python3 -m venv venv
            . venv/bin/activate
            pip install -U pip
            pip install -r requirements.txt
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
            # PYTHONPATH is already set at pipeline level
            pytest -q --maxfail=1 --disable-warnings
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
          // safe quoting: single-quoted Groovy string; shell expands $VARS; JSON is valid
          sh '''
            curl -sS https://webexapis.com/v1/messages \
              -H "Authorization: Bearer $WEBEX_TOKEN" \
              -H "Content-Type: application/json" \
              -d "{\"roomId\":\"$WEBEX_ROOM_ID\",\"markdown\":\"**SUCCESS** :white_check_mark: Job *$JOB_NAME* #$BUILD_NUMBER passed tests.\"}"
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
            curl -sS https://webexapis.com/v1/messages \
              -H "Authorization: Bearer $WEBEX_TOKEN" \
              -H "Content-Type: application/json" \
              -d "{\"roomId\":\"$WEBEX_ROOM_ID\",\"markdown\":\"**FAILURE** :x: Job *$JOB_NAME* #$BUILD_NUMBER failed. Check Jenkins console.\"}"
          '''
        }
      }
    }
  }
}
