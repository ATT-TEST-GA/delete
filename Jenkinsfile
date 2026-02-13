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
      description: 'TRUE = Backup only. FALSE = Allow delete/backout'
    )

    booleanParam(
      name: 'ENABLE_DELETE',
      defaultValue: false,
      description: 'Must be checked to delete branches'
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
    GITHUB_TOKEN       = credentials('github-pat')
    GITHUB_ORG         = 'ATT-TEST-GA'
    MIRROR_BACKUP_DIR  = '/opt/git-mirror-backups'
    ALLOWED_APPROVERS  = 'admin,cloudops,devopslead'
    REPORT_FILE        = 'branch_operation_audit.csv'
  }

  stages {

    stage('Validate Inputs') {
      steps {
        script {
          if (!params.REPO_BRANCH_INPUT?.trim()) {
            error('❌ REPO_BRANCH_INPUT cannot be empty')
          }

          if (!params.DRY_RUN && !params.ENABLE_DELETE && !params.ENABLE_BACKOUT) {
            error('❌ When DRY_RUN=false, either DELETE or BACKOUT must be enabled')
          }

          if (params.ENABLE_DELETE && params.ENABLE_BACKOUT) {
            error('❌ DELETE and BACKOUT cannot run together')
          }

          echo "Parameters validated successfully."
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
          writeFile file: 'approved_by.txt', text: approver

          echo "Approved by: ${approver}"
        }
      }
    }

    stage('Initialize Audit Report') {
      steps {
        sh '''
echo "Timestamp,Repo,Branch,Action,Status" > ${REPORT_FILE}
'''
      }
    }

    stage('Mirror Backup (Real Backup)') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail

mkdir -p "${MIRROR_BACKUP_DIR}"

while IFS= read -r raw || [ -n "$raw" ]; do

  line=$(echo "$raw" | sed 's/|/:/g' | xargs)
  REPO="${line%%:*}"
  BRANCH="${line##*:}"
  TARGET="${MIRROR_BACKUP_DIR}/${REPO}.git"

  echo "📦 Processing backup for $REPO"

  if [ -d "$TARGET" ]; then
    cd "$TARGET"
    git remote update
    cd - >/dev/null
  else
    git clone --mirror \
      https://${GITHUB_TOKEN}@github.com/${GITHUB_ORG}/${REPO}.git \
      "$TARGET"
  fi

  echo "$(date),$REPO,$BRANCH,BACKUP,SUCCESS" >> ${REPORT_FILE}

done < approved.txt

echo "✅ Mirror backup completed"
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

PROTECTED=("main" "master" "develop" "prod")

while IFS= read -r raw || [ -n "$raw" ]; do

  line=$(echo "$raw" | sed 's/|/:/g' | xargs)
  REPO="${line%%:*}"
  BRANCH="${line##*:}"

  for P in "${PROTECTED[@]}"; do
    if [ "$BRANCH" = "$P" ]; then
      echo "$(date),$REPO,$BRANCH,DELETE,BLOCKED_PROTECTED_BRANCH" >> ${REPORT_FILE}
      echo "❌ Cannot delete protected branch: $BRANCH"
      exit 1
    fi
  done

  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    "https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/ref/heads/${BRANCH}")

  if [ "$STATUS" = "404" ]; then
    echo "$(date),$REPO,$BRANCH,DELETE,ALREADY_DELETED" >> ${REPORT_FILE}
    continue
  fi

  curl -s -X DELETE \
    -H "Authorization: Bearer ${GITHUB_TOKEN}" \
    "https://api.github.com/repos/${GITHUB_ORG}/${REPO}/git/refs/heads/${BRANCH}"

  echo "$(date),$REPO,$BRANCH,DELETE,SUCCESS" >> ${REPORT_FILE}

done < approved.txt

echo "🗑 Branch deletion completed"
'''
      }
    }

    stage('Backout (Restore Branch)') {
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

  echo "♻ Restoring $REPO:$BRANCH"

  cd "$TARGET"

  git push \
    https://${GITHUB_TOKEN}@github.com/${GITHUB_ORG}/${REPO}.git \
    refs/heads/${BRANCH}:refs/heads/${BRANCH}

  cd - >/dev/null

  echo "$(date),$REPO,$BRANCH,BACKOUT,RESTORED" >> ${REPORT_FILE}

done < approved.txt

echo "♻ Backout completed"
'''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '*.csv,approved.txt,approved_by.txt',
                       fingerprint: true,
                       allowEmptyArchive: true
    }

    success {
      echo "✅ Branch operation completed successfully"
    }

    failure {
      echo "❌ Pipeline failed — operation halted safely"
    }
  }
}

