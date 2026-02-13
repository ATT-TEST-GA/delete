pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(
      numToKeepStr: '100',
      artifactNumToKeepStr: '100'
    ))
    timeout(time: 60, unit: 'MINUTES')
  }

  parameters {
    choice(
      name: 'OPERATION_MODE',
      choices: ['VALIDATE', 'DELETE', 'BACKOUT'],
      description: 'Select operation mode'
    )

    text(
      name: 'REPO_BRANCH_INPUT',
      defaultValue: '',
      description: 'Enter repo:branch pairs (one per line)'
    )
  }

  environment {
    GITHUB_TOKEN      = credentials('github-pat')
    GITHUB_ORG        = 'ATT-TEST-GA'
    MIRROR_BACKUP_DIR = '/opt/git-mirror-backups'
    ALLOWED_APPROVERS = 'admin,cloudops,devopslead'
    REPORT_FILE       = "github_branch_operation_audit_${BUILD_NUMBER}.csv"
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
echo "Timestamp,JobName,BuildNumber,NodeName,ApprovedBy,Repo,Branch,Action,Status,HTTPStatus,BackupPath" > ${REPORT_FILE}
'''
      }
    }

    stage('Validate All Branches First') {
      steps {
        sh '''
#!/usr/bin/env bash
set -euo pipefail

PROTECTED_STATIC=("main" "master" "develop" "prod" "release")
MISSING=()
PROTECTED_HITS=()

while IFS= read -r raw || [ -n "$raw" ]; do
  line=$(echo "$raw" | sed 's/|/:/g' | xargs)
  REPO="${line%%:*}"
  BRANCH="${line##*:}"

  # Get default branch dynamically
  DEFAULT_BRANCH=$(curl -s \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    "https://api.github.com/repos/${GITHUB_ORG}/${REPO}" | \
    grep '"default_branch"' | cut -d '"' -f4)

  if [ "$BRANCH" = "$DEFAULT_BRANCH" ]; then
    PROTECTED_HITS+=("$REPO:$BRANCH (default branch)")
  fi

  # Static protected check
  for P in "${PROTECTED_STATIC[@]}"; do
    if [ "$BRANCH" = "$P" ]; then
      PROTECTED_HITS+=("$REPO:$BRANCH")
    fi
  done

  if [[ "$BRANCH" == release/* ]]; then
    PROTECTED_HITS+=("$REPO:$BRANCH")
  fi

  # Branch existence check
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    "https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/ref/heads/${BRANCH}")

  if [ "$STATUS" = "404" ]; then
    MISSING+=("$REPO:$BRANCH")
  fi

done < <(echo "${REPO_BRANCH_INPUT}")

if [ ${#PROTECTED_HITS[@]} -gt 0 ]; then
  echo "❌ Protected branches detected:"
  printf '%s\n' "${PROTECTED_HITS[@]}"
  exit 1
fi

if [ ${#MISSING[@]} -gt 0 ]; then
  echo "❌ Missing branches:"
  printf '%s\n' "${MISSING[@]}"
  exit 1
fi

echo "✅ Validation successful."
'''
      }
    }

    stage('Approval Gate') {
      when {
        expression { return params.OPERATION_MODE != 'VALIDATE' }
      }
      steps {
        script {
          def approver = input(
            message: """Approve ${params.OPERATION_MODE} operation:

${params.REPO_BRANCH_INPUT}
""",
            ok: 'APPROVE',
            submitter: env.ALLOWED_APPROVERS,
            submitterParameter: 'APPROVED_BY'
          )
          env.APPROVED_BY = approver
        }
      }
    }

    stage('Backup (DELETE Only)') {
      when {
        expression { return params.OPERATION_MODE == 'DELETE' }
      }
      steps {
        sh '''
#!/usr/bin/env bash
set -euo pipefail

mkdir -p "${MIRROR_BACKUP_DIR}"

while IFS= read -r raw || [ -n "$raw" ]; do
  line=$(echo "$raw" | sed 's/|/:/g' | xargs)
  REPO="${line%%:*}"
  TARGET="${MIRROR_BACKUP_DIR}/${REPO}.git"

  if [ -d "$TARGET" ]; then
    cd "$TARGET"
    git remote update
    cd - >/dev/null
  else
    git clone --mirror \
      https://${GITHUB_TOKEN}@github.com/${GITHUB_ORG}/${REPO}.git \
      "$TARGET"
  fi

  # Ensure backup exists
  if [ ! -d "$TARGET" ]; then
    echo "❌ Backup verification failed"
    exit 1
  fi

done < <(echo "${REPO_BRANCH_INPUT}")
'''
      }
    }

    stage('Execute Operation') {
      when {
        expression { return params.OPERATION_MODE != 'VALIDATE' }
      }
      steps {
        sh '''
#!/usr/bin/env bash
set -euo pipefail

MODE="${OPERATION_MODE}"

while IFS= read -r raw || [ -n "$raw" ]; do
  line=$(echo "$raw" | sed 's/|/:/g' | xargs)
  REPO="${line%%:*}"
  BRANCH="${line##*:}"
  TARGET="${MIRROR_BACKUP_DIR}/${REPO}.git"

  if [ "$MODE" = "DELETE" ]; then
    HTTP_STATUS=$(curl -s -o response.json -w "%{http_code}" \
      -X DELETE \
      -H "Authorization: Bearer ${GITHUB_TOKEN}" \
      "https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/refs/heads/${BRANCH}")

    if [ "$HTTP_STATUS" != "204" ]; then
      cat response.json
      echo "$(date),${JOB_NAME},${BUILD_NUMBER},${NODE_NAME},${APPROVED_BY:-SYSTEM},$REPO,$BRANCH,DELETE,FAILED,$HTTP_STATUS,$TARGET" >> ${REPORT_FILE}
      exit 1
    fi

    STATUS_LABEL="DELETED"

  else
    cd "$TARGET"
    git push \
      https://${GITHUB_TOKEN}@github.com/${GITHUB_ORG}/${REPO}.git \
      refs/heads/${BRANCH}:refs/heads/${BRANCH}
    cd - >/dev/null
    STATUS_LABEL="RESTORED"
    HTTP_STATUS="200"
  fi

  echo "$(date),${JOB_NAME},${BUILD_NUMBER},${NODE_NAME},${APPROVED_BY:-SYSTEM},$REPO,$BRANCH,$MODE,$STATUS_LABEL,$HTTP_STATUS,$TARGET" >> ${REPORT_FILE}

done < <(echo "${REPO_BRANCH_INPUT}")
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
