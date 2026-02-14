pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '30'))
    timeout(time: 60, unit: 'MINUTES')
  }

  parameters {
    choice(
      name: 'OPERATION_MODE',
      choices: ['DRY_RUN', 'DELETE'],
      description: 'Select operation mode'
    )

    text(
      name: 'REPO_BRANCH_INPUT',
      defaultValue: '',
      description: '''Enter repo:branch pairs (one per line)
Example:
APM0012058-TEST:feature/test
'''
    )
  }

  environment {
    GITHUB_TOKEN      = credentials('github-pat')
    GITHUB_ORG        = 'ATT-TEST-GA'
    ALLOWED_APPROVERS = 'admin,cloudops,devopslead'
    REPORT_FILE       = 'branch_operation_audit.csv'
  }

  stages {

    stage('Validate Parameters') {
      steps {
        script {
          if (!params.REPO_BRANCH_INPUT?.trim()) {
            error('❌ REPO_BRANCH_INPUT cannot be empty')
          }
        }
      }
    }

    stage('Initialize Audit Report') {
      steps {
        sh '''
echo "Timestamp,JobName,BuildNumber,NodeName,ApprovedBy,Repo,Branch,Operation,Status" > ${REPORT_FILE}
'''
      }
    }

    stage('Enterprise Validation') {
      steps {
        sh '''
set -eu

PROTECTED_EXACT="main master develop dev prod production uat qa stage staging"
PROTECTED_PREFIX="release/ hotfix/ support/"

PROTECTED_HITS=""
MISSING=""

while read line; do
  line=$(echo "$line" | xargs)
  [ -z "$line" ] && continue

  REPO=${line%%:*}
  BRANCH=${line##*:}
  BRANCH_LOWER=$(echo "$BRANCH" | tr '[:upper:]' '[:lower:]')

  # Fetch default branch
  DEFAULT_BRANCH=$(curl -s \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    https://api.github.com/repos/${GITHUB_ORG}/${REPO} \
    | grep -o '"default_branch": *"[^"]*"' \
    | cut -d'"' -f4)

  if [ "$BRANCH" = "$DEFAULT_BRANCH" ]; then
    echo "❌ Cannot delete default branch: $REPO:$BRANCH"
    exit 1
  fi

  # Exact protection
  for P in $PROTECTED_EXACT; do
    if [ "$BRANCH_LOWER" = "$P" ]; then
      PROTECTED_HITS="$PROTECTED_HITS $REPO:$BRANCH"
    fi
  done

  # Prefix protection
  for P in $PROTECTED_PREFIX; do
    case "$BRANCH_LOWER" in
      $P*) PROTECTED_HITS="$PROTECTED_HITS $REPO:$BRANCH" ;;
    esac
  done

  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/ref/heads/${BRANCH})

  if [ "$STATUS" = "404" ]; then
    MISSING="$MISSING $REPO:$BRANCH"
  fi

done <<EOF
${REPO_BRANCH_INPUT}
EOF

if [ -n "$PROTECTED_HITS" ]; then
  echo "❌ Protected branches detected:$PROTECTED_HITS"
  exit 1
fi

if [ -n "$MISSING" ]; then
  echo "❌ Missing branches detected:$MISSING"
  exit 1
fi

echo "✅ Enterprise validation successful."
'''
      }
    }

    stage('Approval Gate (DELETE Only)') {
      when {
        expression { params.OPERATION_MODE == 'DELETE' }
      }
      steps {
        script {
          input(
            message: """🚨 PRODUCTION DELETE APPROVAL REQUIRED

${params.REPO_BRANCH_INPUT}
""",
            ok: 'APPROVE DELETE',
            submitter: env.ALLOWED_APPROVERS
          )
        }
      }
    }

    stage('Delete Execution') {
      when {
        expression { params.OPERATION_MODE == 'DELETE' }
      }
      steps {
        sh '''
set -eu

while read line; do
  line=$(echo "$line" | xargs)
  [ -z "$line" ] && continue

  REPO=${line%%:*}
  BRANCH=${line##*:}

  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
    -X DELETE \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    -H "Accept: application/vnd.github+json" \
    https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/refs/heads/${BRANCH})

  if [ "$HTTP_CODE" = "204" ]; then
    STATUS="DELETED"
  elif [ "$HTTP_CODE" = "404" ]; then
    echo "❌ Branch not found: $REPO:$BRANCH"
    exit 1
  elif [ "$HTTP_CODE" = "403" ]; then
    echo "❌ Permission denied: $REPO:$BRANCH"
    exit 1
  elif [ "$HTTP_CODE" = "422" ]; then
    echo "❌ GitHub rejected deletion (422). Possibly protected or default branch: $REPO:$BRANCH"
    exit 1
  else
    echo "❌ GitHub API failed with HTTP $HTTP_CODE for $REPO:$BRANCH"
    exit 1
  fi

  echo "$(date),${JOB_NAME},${BUILD_NUMBER},${NODE_NAME},SYSTEM,$REPO,$BRANCH,DELETE,${STATUS}" >> ${REPORT_FILE}

done <<EOF
${REPO_BRANCH_INPUT}
EOF

echo "✅ Delete completed successfully."
'''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '*.csv',
                       fingerprint: true,
                       allowEmptyArchive: true
    }
  }
}
