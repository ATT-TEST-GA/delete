pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '30'))
    timeout(time: 45, unit: 'MINUTES')
  }

  parameters {
    booleanParam(
      name: 'DRY_RUN',
      defaultValue: true,
      description: 'TRUE = Validation only. FALSE = Allow delete/backout'
    )

    booleanParam(
      name: 'ENABLE_DELETE',
      defaultValue: false,
      description: 'Enable branch deletion'
    )

    booleanParam(
      name: 'ENABLE_BACKOUT',
      defaultValue: false,
      description: 'Restore branch from mirror backup'
    )

    text(
      name: 'REPO_BRANCH_INPUT',
      defaultValue: '',
      description: '''Enter repo:branch pairs (one per line)

Example:
APM0012058-TEST:feature/test1
APM0014540-INFRASTRUCTURE:bugfix/cleanup
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

          if (!params.DRY_RUN && !params.ENABLE_DELETE && !params.ENABLE_BACKOUT) {
            error('❌ When DRY_RUN=false, enable DELETE or BACKOUT')
          }

          if (params.ENABLE_DELETE && params.ENABLE_BACKOUT) {
            error('❌ DELETE and BACKOUT cannot run together')
          }
        }
      }
    }

    stage('Initialize Audit Report') {
      steps {
        sh '''
echo "Timestamp,JobName,BuildNumber,Repo,Branch,Action,Status" > ${REPORT_FILE}
'''
      }
    }

    stage('Validate All Branches First') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail

PROTECTED=("main" "master" "develop" "prod" "release")

MISSING=()
PROTECTED_HITS=()

while IFS= read -r raw || [ -n "$raw" ]; do
  line=$(echo "$raw" | sed 's/|/:/g' | xargs)
  REPO="${line%%:*}"
  BRANCH="${line##*:}"

  # Protected check
  for P in "${PROTECTED[@]}"; do
    if [ "$BRANCH" = "$P" ]; then
      PROTECTED_HITS+=("$REPO:$BRANCH")
    fi
  done

  if [[ "$BRANCH" == release/* ]]; then
    PROTECTED_HITS+=("$REPO:$BRANCH")
  fi

  # Existence check
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    "https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/ref/heads/${BRANCH}")

  if [ "$STATUS" = "404" ]; then
    MISSING+=("$REPO:$BRANCH")
  fi

done < <(echo "${REPO_BRANCH_INPUT}")

if [ ${#PROTECTED_HITS[@]} -gt 0 ]; then
  echo "❌ Protected branches:"
  printf '%s\n' "${PROTECTED_HITS[@]}"
  exit 1
fi

if [ ${#MISSING[@]} -gt 0 ]; then
  echo "❌ Missing branches:"
  printf '%s\n' "${MISSING[@]}"
  exit 1
fi

echo "✅ All branches validated successfully."
'''
      }
    }

    stage('Approval Gate') {
      when {
        expression { return !params.DRY_RUN }
      }
      steps {
        script {
          input(
            message: """Approve branch operation:

${params.REPO_BRANCH_INPUT}

DELETE: ${params.ENABLE_DELETE}
BACKOUT: ${params.ENABLE_BACKOUT}
""",
            ok: 'APPROVE',
            submitter: env.ALLOWED_APPROVERS
          )
        }
      }
    }

    stage('Backup (Only When Deleting)') {
      when {
        expression { return !params.DRY_RUN && params.ENABLE_DELETE }
      }
      steps {
        sh '''#!/usr/bin/env bash
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

done < <(echo "${REPO_BRANCH_INPUT}")
'''
      }
    }

    stage('Delete Branches') {
      when {
        expression { return !params.DRY_RUN && params.ENABLE_DELETE }
      }
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail

while IFS= read -r raw || [ -n "$raw" ]; do
  line=$(echo "$raw" | sed 's/|/:/g' | xargs)
  REPO="${line%%:*}"
  BRANCH="${line##*:}"

  curl -s -X DELETE \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    "https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/refs/heads/${BRANCH}"

  echo "$(date),${JOB_NAME},${BUILD_NUMBER},$REPO,$BRANCH,DELETE,SUCCESS" >> ${REPORT_FILE}
  echo "Deleted $REPO:$BRANCH"

done < <(echo "${REPO_BRANCH_INPUT}")
'''
      }
    }

    stage('Backout') {
      when {
        expression { return !params.DRY_RUN && params.ENABLE_BACKOUT }
      }
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail

while IFS= read -r raw || [ -n "$raw" ]; do
  line=$(echo "$raw" | sed 's/|/:/g' | xargs)
  REPO="${line%%:*}"
  BRANCH="${line##*:}"
  TARGET="${MIRROR_BACKUP_DIR}/${REPO}.git"

  cd "$TARGET"

  git push \
    https://${GITHUB_TOKEN}@github.com/${GITHUB_ORG}/${REPO}.git \
    refs/heads/${BRANCH}:refs/heads/${BRANCH}

  cd - >/dev/null

  echo "$(date),${JOB_NAME},${BUILD_NUMBER},$REPO,$BRANCH,BACKOUT,RESTORED" >> ${REPORT_FILE}
  echo "Restored $REPO:$BRANCH"

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
