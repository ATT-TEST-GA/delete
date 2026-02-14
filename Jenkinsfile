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
        sh(script: '''
set -euo pipefail

PROTECTED_EXACT=("main" "master" "develop" "dev" "prod" "production" "uat" "qa" "stage" "staging")
PROTECTED_PREFIX=("release/" "hotfix/" "support/")

declare -A SEEN
PROTECTED_HITS=()
MISSING=()

while read line; do
  line=$(echo "$line" | xargs)
  [ -z "$line" ] && continue

  REPO="${line%%:*}"
  BRANCH="${line##*:}"
  KEY="${REPO}:${BRANCH}"

  if [[ -n "${SEEN[$KEY]:-}" ]]; then
    continue
  fi
  SEEN[$KEY]=1

  BRANCH_LOWER=$(echo "$BRANCH" | tr '[:upper:]' '[:lower:]')

  # Protected exact
  for P in "${PROTECTED_EXACT[@]}"; do
    if [[ "$BRANCH_LOWER" == "$P" ]]; then
      PROTECTED_HITS+=("$REPO:$BRANCH")
    fi
  done

  # Protected prefix
  for P in "${PROTECTED_PREFIX[@]}"; do
    if [[ "$BRANCH_LOWER" == ${P}* ]]; then
      PROTECTED_HITS+=("$REPO:$BRANCH")
    fi
  done

  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    "https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/ref/heads/${BRANCH}")

  if [ "$STATUS" = "404" ]; then
    MISSING+=("$REPO:$BRANCH")
  fi

done <<< "${REPO_BRANCH_INPUT}"

if [ ${#PROTECTED_HITS[@]} -gt 0 ]; then
  echo "❌ Protected branches detected:"
  printf '%s\n' "${PROTECTED_HITS[@]}"
  exit 1
fi

if [ ${#MISSING[@]} -gt 0 ]; then
  echo "❌ Missing branches detected:"
  printf '%s\n' "${MISSING[@]}"
  exit 1
fi

echo "✅ Validation successful."
''', shell: '/bin/bash')
      }
    }

    stage('Approval Gate (DELETE Only)') {
      when {
        expression { params.OPERATION_MODE == 'DELETE' }
      }
      steps {
        script {
          input(
            message: """PRODUCTION DELETE APPROVAL REQUIRED

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
        sh(script: '''
set -euo pipefail

while read line; do
  line=$(echo "$line" | xargs)
  [ -z "$line" ] && continue

  REPO="${line%%:*}"
  BRANCH="${line##*:}"

  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
    -X DELETE \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    -H "Accept: application/vnd.github+json" \
    "https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/refs/heads/${BRANCH}")

  if [ "$HTTP_CODE" = "204" ]; then
    STATUS="DELETED"
  elif [ "$HTTP_CODE" = "404" ]; then
    echo "❌ Branch not found: $REPO:$BRANCH"
    exit 1
  elif [ "$HTTP_CODE" = "403" ]; then
    echo "❌ Permission denied: $REPO:$BRANCH"
    exit 1
  else
    echo "❌ GitHub API failed with HTTP $HTTP_CODE for $REPO:$BRANCH"
    exit 1
  fi

  echo "$(date),${JOB_NAME},${BUILD_NUMBER},${NODE_NAME},SYSTEM,$REPO,$BRANCH,DELETE,${STATUS}" >> ${REPORT_FILE}

done <<< "${REPO_BRANCH_INPUT}"

echo "✅ Delete completed."
''', shell: '/bin/bash')
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
