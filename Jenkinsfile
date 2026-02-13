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

    stage('Approval Gate') {
      steps {
        script {
          def approver = input(
            message: """Approve branch operation:

${params.REPO_BRANCH_INPUT}

DRY_RUN: ${params.DRY_RUN}
DELETE: ${params.ENABLE_DELETE}
BACKOUT: ${params.ENABLE_BACKOUT}
""",
            ok: 'APPROVE',
            submitter: env.ALLOWED_APPROVERS,
            submitterParameter: 'APPROVED_BY'
          )

          writeFile file: 'approved.txt', text: params.REPO_BRANCH_INPUT.trim()
          env.APPROVED_BY = approver
        }
      }
    }

    stage('Initialize Audit Report') {
      steps {
        sh '''
echo "Timestamp,JobName,BuildNumber,ApprovedBy,Repo,Branch,Action,Status,BackupPath" > ${REPORT_FILE}
'''
      }
    }

    stage('Validate All Branches First') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail

PROTECTED=("main" "master" "develop" "prod" "release")

MISSING_BRANCHES=()
PROTECTED_BRANCHES=()

while IFS= read -r raw || [ -n "$raw" ]; do

  line=$(echo "$raw" | sed 's/|/:/g' | xargs)
  REPO="${line%%:*}"
  BRANCH="${line##*:}"

  # ---- Protected check ----
  for P in "${PROTECTED[@]}"; do
    if [ "$BRANCH" = "$P" ]; then
      PROTECTED_BRANCHES+=("$REPO:$BRANCH")
    fi
  done

  if [[ "$BRANCH" == release/* ]]; then
      PROTECTED_BRANCHES+=("$REPO:$BRANCH")
  fi

  # ---- Branch existence check ----
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    "https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/ref/heads/${BRANCH}")

  if [ "$STATUS" = "404" ]; then
    MISSING_BRANCHES+=("$REPO:$BRANCH")
  fi

done < approved.txt

if [ ${#PROTECTED_BRANCHES[@]} -gt 0 ]; then
  echo "❌ Protected branches detected:"
  printf '%s\n' "${PROTECTED_BRANCHES[@]}"
  exit 1
fi

if [ ${#MISSING_BRANCHES[@]} -gt 0 ]; then
  echo "❌ The following branches do NOT exist:"
  printf '%s\n' "${MISSING_BRANCHES[@]}"
  exit 1
fi

echo "✅ All branches validated successfully."
'''
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

done < approved.txt
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
  TARGET="${MIRROR_BACKUP_DIR}/${REPO}.git"

  curl -s -X DELETE \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    "https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/refs/heads/${BRANCH}"

  echo "$(date),${JOB_NAME},${BUILD_NUMBER},${APPROVED_BY},$REPO,$BRANCH,DELETE,SUCCESS,$TARGET" >> ${REPORT_FILE}

done < approved.txt
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

  echo "$(date),${JOB_NAME},${BUILD_NUMBER},${APPROVED_BY},$REPO,$BRANCH,BACKOUT,RESTORED,$TARGET" >> ${REPORT_FILE}

done < approved.txt
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
