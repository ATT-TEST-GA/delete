pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '30'))
    timeout(time: 60, unit: 'MINUTES')
  }

  parameters {
    booleanParam(
      name: 'DRY_RUN',
      defaultValue: true,
      description: 'TRUE = Validation only. FALSE = Request delete'
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
    MIRROR_BACKUP_DIR = '/opt/git-mirror-backups'
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
echo "Timestamp,JobName,BuildNumber,NodeName,ApprovedBy,Repo,Branch,Action,Status,BackupTaken" > ${REPORT_FILE}
'''
      }
    }

    stage('Enterprise Validation') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail

PROTECTED_EXACT=("main" "master" "develop" "dev" "prod" "production" "uat" "qa" "stage" "staging" "head")
PROTECTED_PREFIX=("release/" "hotfix/" "support/")

declare -A SEEN
PROTECTED_HITS=()
MISSING=()

while IFS= read -r raw || [ -n "$raw" ]; do
  line=$(echo "$raw" | xargs)
  [ -z "$line" ] && continue

  REPO="${line%%:*}"
  BRANCH="${line##*:}"
  KEY="${REPO}:${BRANCH}"

  if [[ -n "${SEEN[$KEY]:-}" ]]; then
    continue
  fi
  SEEN[$KEY]=1

  BRANCH_LOWER=$(echo "$BRANCH" | tr '[:upper:]' '[:lower:]')

  for P in "${PROTECTED_EXACT[@]}"; do
    if [[ "$BRANCH_LOWER" == "$P" ]]; then
      PROTECTED_HITS+=("$REPO:$BRANCH")
    fi
  done

  for P in "${PROTECTED_PREFIX[@]}"; do
    if [[ "$BRANCH_LOWER" == ${P}* ]]; then
      PROTECTED_HITS+=("$REPO:$BRANCH")
    fi
  done

  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    -H "Accept: application/vnd.github+json" \
    "https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/ref/heads/${BRANCH}")

  if [ "$STATUS" = "404" ]; then
    MISSING+=("$REPO:$BRANCH")
  fi

done < <(echo "${REPO_BRANCH_INPUT}")

if [ ${#PROTECTED_HITS[@]} -gt 0 ]; then
  echo "❌ Attempt to delete protected branches:"
  printf '%s\n' "${PROTECTED_HITS[@]}"
  exit 1
fi

if [ ${#MISSING[@]} -gt 0 ]; then
  echo "❌ Missing branches detected:"
  printf '%s\n' "${MISSING[@]}"
  exit 1
fi

echo "✅ Enterprise validation successful."
'''
      }
    }

    stage('Approval Gate (Governed Decision)') {
      when {
        expression { return !params.DRY_RUN }
      }
      steps {
        script {
          def decision = input(
            message: """PRODUCTION DELETE APPROVAL REQUIRED

${params.REPO_BRANCH_INPUT}

Select action:
""",
            ok: 'PROCEED',
            submitter: env.ALLOWED_APPROVERS,
            parameters: [
              choice(
                name: 'DELETE_MODE',
                choices: ['BACKUP_AND_DELETE', 'DIRECT_DELETE'],
                description: 'Select deletion mode'
              )
            ]
          )

          env.DELETE_MODE = decision
        }
      }
    }

    stage('Backup (If Selected)') {
      when {
        expression {
          return !params.DRY_RUN &&
                 env.DELETE_MODE == 'BACKUP_AND_DELETE'
        }
      }
      steps {
        sh '''
set -euo pipefail
mkdir -p "${MIRROR_BACKUP_DIR}"

while read line; do
  line=$(echo "$line" | xargs)
  [ -z "$line" ] && continue

  REPO="${line%%:*}"
  TARGET="${MIRROR_BACKUP_DIR}/${REPO}.git"

  if [ -d "$TARGET" ]; then
    cd "$TARGET" && git remote update && cd - >/dev/null
  else
    git clone --mirror \
      https://${GITHUB_TOKEN}@github.com/${GITHUB_ORG}/${REPO}.git \
      "$TARGET"
  fi

done <<< "${REPO_BRANCH_INPUT}"

echo "✅ Backup completed."
'''
      }
    }

    stage('Delete Execution (With HTTP Handling)') {
      when {
        expression { return !params.DRY_RUN }
      }
      steps {
        sh '''
set -euo pipefail

BACKUP_STATUS="NO"
[ "${DELETE_MODE}" = "BACKUP_AND_DELETE" ] && BACKUP_STATUS="YES"

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

  echo "$(date),${JOB_NAME},${BUILD_NUMBER},${NODE_NAME},${APPROVED_BY:-SYSTEM},$REPO,$BRANCH,DELETE,${STATUS},${BACKUP_STATUS}" >> ${REPORT_FILE}

done <<< "${REPO_BRANCH_INPUT}"

echo "✅ Production delete completed."
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
