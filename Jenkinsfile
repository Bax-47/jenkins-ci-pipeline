pipeline {
  agent any
  options { timestamps() }

  environment {
    // So "from app..." works in pytest
    PYTHONPATH = "${WORKSPACE}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Setup Python + deps') {
      steps {
        ansiColor('xterm') {
          sh '''
            set -eux
            python3 --version
            python3 -m venv venv
            . venv/bin/activate
            # Use venv's python/pip only (avoids PEP 668 issues)
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
            mkdir -p reports
            # JUnit + Cobertura XML coverage + terminal summary
            venv/bin/python -m pytest -q --maxfail=1 --disable-warnings \
              --junitxml=reports/junit.xml \
              --cov=app --cov-report=xml:reports/coverage.xml --cov-report=term-missing
          '''
        }
      }
    }
  }

  post {
    always {
      // Publish tests & artifacts
      junit 'reports/junit.xml'
      archiveArtifacts artifacts: 'reports/**', fingerprint: true

      // Publish coverage using Code Coverage API (requires "Code Coverage API" + "Cobertura" parser)
      script {
        if (fileExists('reports/coverage.xml')) {
          recordCoverage tools: [cobertura(pattern: 'reports/coverage.xml')],
                         sourceFileResolver: sourceFiles('STORE_LAST_BUILD'),
                         calculateDiffForChangeRequests: true,
                         failNoReports: false
        } else {
          echo 'Coverage file not found (reports/coverage.xml) — skipping coverage publish.'
        }
      }
    }

    success {
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
