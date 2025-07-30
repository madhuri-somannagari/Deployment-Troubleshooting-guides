## Automated Application Deployment to VM using GitHub Actions (with Rollback Support)

### Concept Overview

This project automates the deployment of a web application to a virtual machine (VM) using **GitHub Actions**. The solution:

- Delivers seamless application updates from code commits to the live environment.
- Minimizes manual intervention and deployment errors.
- Implements a **rollback mechanism** to revert to the last stable state in case of failure.
- Ensures high availability and faster recovery, especially for production environments.

---

### Objective

To implement a **robust and reliable CI/CD pipeline** using:
- **GitHub Actions** for automation,
- **Rsync** for efficient file transfers,
- **Systemd-managed services** like **Gunicorn** and **Celery**,
- **Rollback support** in case of deployment issues,
- Log management with **logrotate**,
- Conditional service restarts based on code changes.

---

### Architecture Overview

```text
Developer Pushes Code → GitHub Actions Workflow (.github/workflows/deploy.yml)
                         ↓
                  Self-Hosted Runner on VM (EC2)
                         ↓
                Bash Scripts: deploy.sh / rollback_latest.sh
                         ↓
        Gunicorn, Nginx, Celery, and Celery Beat Restarted as Needed
                         ↓
            Slack Notifications for Success or Rollback
```

### Deployment Workflow
1. Developer pushes code to main or master.
2. GitHub Actions picks up the push via self-hosted runner.
3. deploy.sh is triggered by workflow and performs:
  - Create timestamped directory in /home/ubuntu/releases/<timestamp>
  - Copy source code from /home/ubuntu/git-source (not directly from .git)
  - Set up or reuse shared Python virtual environment
  - Install dependencies if requirements.txt changed
  - Apply Django database migrations
  - Copy .env file and link latest release with: `ln -sfn /home/ubuntu/releases/<timestamp> /home/ubuntu/current`
  - Restart services (Gunicorn, Celery, etc.)
4. If deployment fails:
  - rollback_latest.sh is triggered
  - Restores symlink to the previous release
  - Restarts services to revert to stable state
5. Slack Notifications:
  - ✅ Success → "Deployment Successful"
  - ❌ Failure → "Deployment Failed - Rollback Triggered"

## Repository Structure
```
/
├── .github/
│   └── workflows/
│       └── deploy.yml         # GitHub Actions CI/CD pipeline
├── scripts/
│   ├── deploy.sh              # Deployment script
│   └── rollback_latest.sh     # Rollback script
├── README.md

```

## Virtual Machine Setup
Script Name	|Description	| Path
deploy.sh	  |Handles deployment with zero downtime	|/home/ubuntu/deploy.sh
rollback_latest.sh	|Handles rollback to last stable release	|/home/ubuntu/rollback_latest.sh

permissions:
``` bash
chmod +x /home/ubuntu/deploy.sh
chmod +x /home/ubuntu/rollback_latest.sh
```

### Deployment Strategy 
1. Efficient Code Sync
``` bash.sh
rsync -a \
  --exclude '.git' \
  --exclude '__pycache__' \
  --exclude '*.pyc' \
  --exclude '.pytest_cache' \
  "$REPO_DIR/" "$RELEASE_DIR/"
```
2. Conditional Dependency Installation Only install Python dependencies if requirements.txt has changed:
   ``` bash.sh
   cmp -s "$CURRENT_RELEASE/requirements.txt" "$RELEASE_DIR/requirements.txt" || pip install -r requirements.txt
   ```
3. Shared Virtual Environment
  - Reuse shared_venv across deployments to save time and space.
4. Django App Lifecycle
 - python manage.py migrate
 - python manage.py check --deploy
 - Health check endpoint or management command
5. Zero Downtime Reload (Gunicorn)
 ``` bash.sh
  kill -USR2 <gunicorn_master_pid>  # start new master
  kill -WINCH <old_master_pid>      # gracefully stop old workers
  kill -TERM <old_master_pid>       # stop old master
```
6. Selective Celery Restart
 Restart only if celery/, tasks/, or related files have changed:
``` bash.sh
systemctl restart celery
systemctl restart celery-beat
```
7. Old Release Cleanup
  Keep only the last 5 releases: `` ls -dt /home/ubuntu/releases/* | tail -n +6 | xargs rm -rf ``

8. Rollback Strategy
Script: ` rollback_latest.sh `
Rollback Logic:
   - Identify the last successful release (based on timestamp)
   - Point current symlink to that release
   - Restart services to revert to previous state.

### deploy.sh
```script.sh
#!/bin/bash
#-e-errexit, -u-unset variables exit, -o-pipefail exit if one command fails
set -euo pipefail

# ===== Configurable Variables =====
BRANCH="${1:-staging-production}"

REPO_DIR="/home/ubuntu/git-source"
APP_DIR="/home/ubuntu"
RELEASES_DIR="$APP_DIR/releases"
TIMESTAMP=$(date +%Y%m%d%H%M%S)
RELEASE_DIR="$RELEASES_DIR/$TIMESTAMP"
SHARED_VENV="/home/ubuntu/shared_venv"
CURRENT_LINK="$APP_DIR/Hiringdog-backend"
REQ_HASH_FILE="$SHARED_VENV/.last_requirements.hash"

log() {
    echo "[`date +%Y-%m-%d\ %H:%M:%S`] $1"
}

# ===== Pull Code =====
log "Pulling latest code from branch: $BRANCH"
cd "$REPO_DIR"
git fetch origin
git checkout "$BRANCH"
git pull origin "$BRANCH"

# ===== Prepare Release Folder =====
log "Creating release directory: $RELEASE_DIR"
mkdir -p "$RELEASE_DIR"

log "Copying project files..."
rsync -a \
  --exclude '.git' \
  --exclude '__pycache__' \
  --exclude '*.pyc' \
  --exclude '.pytest_cache' \
  "$REPO_DIR/" "$RELEASE_DIR/"


# ===== Set up Python Environment =====
cd "$RELEASE_DIR"

log "Setting up virtual environment..."

if [ ! -d "$SHARED_VENV" ]; then
    log "Creating shared virtual environment at $SHARED_VENV"
    python3.12 -m venv "$SHARED_VENV"
fi
log "Linking shared virtual environment..."
ln -sfn "$SHARED_VENV" "$RELEASE_DIR/venv"

log "Activating virtual environment..."
source "$RELEASE_DIR/venv/bin/activate"

log "Installing dependencies..."

REQ_HASH=$(sha256sum requirements.txt | awk '{print $1}')
if [ ! -f "$REQ_HASH_FILE" ] || [ "$(cat $REQ_HASH_FILE)" != "$REQ_HASH" ]; then
    log "requirements.txt changed. Installing dependencies..."
    pip install --upgrade pip
    pip install --no-cache-dir -r requirements.txt
    echo "$REQ_HASH" > "$REQ_HASH_FILE"
else
    log "No changes in requirements.txt. Skipping pip install."
fi

# ====Environment and Django checks =====
log "Checking .env file..."
if [ ! -f ".env" ]; then
    log "[ERROR] .env file not found. Deployment aborted."
    exit 1
fi

# ===== Run Migrations and Health Checks =====
log "Running migrations..."
set -a && source .env && set +a
python manage.py migrate --noinput

log "Performing Django health check..."
if ! python manage.py check; then
    log "[ERROR] Django check failed. Deployment aborted."
    exit 1
fi

log "Checking for unapplied migrations..."
if ! python manage.py makemigrations --check --dry-run; then
    log "[ERROR] Uncommitted model changes detected. Deployment aborted."
    exit 1
fi

# ===== Update Symlink =====
log "Updating symlink: $CURRENT_LINK -> $RELEASE_DIR"
ln -sfn "$RELEASE_DIR" "$CURRENT_LINK"


# ===== Restart Services =====
log "Reloading/restarting services..."

if systemctl is-active --quiet gunicorn; then
    log "Performing zero-downtime Gunicorn restart..."
    
    # Get the master process PID
    GUNICORN_PID=$(systemctl show --property MainPID gunicorn | cut -d= -f2)
    
    if [[ "$GUNICORN_PID" != "0" ]]; then
        # Send USR2 to start new master with new workers
        sudo kill -USR2 "$GUNICORN_PID"
        
        # Wait for new master to start
        sleep 3
        
        # Send WINCH to old master to gracefully shut down old workers
        sudo kill -WINCH "$GUNICORN_PID"
        
        # Wait a bit more for graceful shutdown
        sleep 2
        
        # Send TERM to old master to shut it down completely
        sudo kill -TERM "$GUNICORN_PID"
        
        log "Zero-downtime Gunicorn restart completed"
    else
        log "Could not get Gunicorn PID, falling back to systemctl restart"
        sudo systemctl restart gunicorn
    fi
else
    log "Starting Gunicorn..."
    sudo systemctl start gunicorn
fi

# Graceful restart of Celery services
log "Checking if Celery restart is needed..."

CELERY_SHOULD_RESTART=false

# Read the last deployed Git commit hash from a file (if it exists).
# This helps detect what's changed since the last deployment.
LAST_DEPLOYED_COMMIT=$(cat "$APP_DIR/.last_deployed_commit" 2>/dev/null || echo "")

# Temporarily move into the application repo directory to run Git commands
pushd "$REPO_DIR" > /dev/null

CURRENT_COMMIT=$(git rev-parse HEAD)

# If we have a last deployed commit, compare it to the current commit and get a list of files that changed.
if [ -n "$LAST_DEPLOYED_COMMIT" ]; then
    CHANGED_FILES=$(git diff --name-only "$LAST_DEPLOYED_COMMIT" "$CURRENT_COMMIT")
else
    CHANGED_FILES=$(git diff --name-only HEAD~5)  # fallback
fi

#returns to the previous directory
popd > /dev/null

if echo "$CHANGED_FILES" | grep -E 'tasks\.py|celery(\.py|/)|requirements\.txt'; then
    log "Restarting Celery and Celery Beat..."
    sudo systemctl restart celery
    sudo systemctl restart celery-beat
else
    log "No changes affecting Celery. Skipping restart."
fi

echo "$CURRENT_COMMIT" > "$APP_DIR/.last_deployed_commit"

log "[SUCCESS] Deployment complete: $RELEASE_DIR"

# ===== Clean Up Old Releases =====
log "Cleaning up old releases (keeping last 5)..."
cd "$RELEASES_DIR"
ls -1dt "$RELEASES_DIR"/* | tail -n +6 | xargs -r rm -rf

exit 0
```
### rollback_latest.sh
```
#!/bin/bash
set -euo pipefail

log() {
    echo "[`date +%Y-%m-%d\ %H:%M:%S`] $1"
}

RELEASES_DIR="/home/ubuntu/releases"
SYMLINK="/home/ubuntu/Hiringdog-backend"

RELEASES=($(ls -td ${RELEASES_DIR}/*))

if [ ${#RELEASES[@]} -lt 2 ]; then
    log "[ERROR] Not enough releases to perform rollback."
    exit 1
fi

CURRENT=${RELEASES[0]}
PREVIOUS=${RELEASES[1]}

if [ ! -d "$PREVIOUS" ]; then
    log "[ERROR] Previous release directory does not exist: $PREVIOUS"
    exit 1
fi

log "Current symlink points to: $(readlink -f $SYMLINK)"
log "Rolling back to previous release: $PREVIOUS"

# Update the symlink atomically
ln -sfn "$PREVIOUS" "$SYMLINK"

# Restart services
log "Restarting Celery services..."
if ! sudo systemctl restart celery; then
    log "[ERROR] Failed to restart Celery."
    exit 1
fi
if ! sudo systemctl restart celery-beat; then
    log "[ERROR] Failed to restart Celery Beat."
    exit 1
fi

log "Reloading/restarting Gunicorn..."
if systemctl is-active --quiet gunicorn; then
    log "Performing zero-downtime Gunicorn restart..."
    
    # Get the master process PID
    GUNICORN_PID=$(systemctl show --property MainPID gunicorn | cut -d= -f2)
    
    if [[ "$GUNICORN_PID" != "0" ]]; then
        # Send USR2 to start new master with new workers
        sudo kill -USR2 "$GUNICORN_PID" 

        # Wait for new master to start
        sleep 3      

        # Send WINCH to old master to gracefully shut down old workers
        sudo kill -WINCH "$GUNICORN_PID"

        # Wait a bit more for graceful shutdown
        sleep 2
        
        # Send TERM to old master to shut it down completely
        sudo kill -TERM "$GUNICORN_PID"
        
        log "Zero-downtime Gunicorn restart completed"
    else
        log "Could not get Gunicorn PID, falling back to systemctl restart"
        sudo systemctl restart gunicorn
    fi
else
    log "Starting Gunicorn..."
    sudo systemctl start gunicorn
fi

log "[SUCCESS] Rollback complete."
log "New symlink points to: $(readlink -f $SYMLINK)"
```

### .github/workflows/staging-deploy.yml
```
name: Deploy to VM via Self-hosted Runner

on:
  push:
    branches: [staging-production]

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run deployment script on VM
        run: |
          sudo -u ubuntu /home/ubuntu/deploy.sh

      - name: Rollback if Deployment Fails
        if: failure()
        run: |
          echo "Deployment failed. Rolling back..."
          sudo -u ubuntu /home/ubuntu/rollback_latest.sh

      - name: Check service status
        run: |
          systemctl status gunicorn --no-pager
          systemctl status celery --no-pager
          systemctl status celery-beat --no-pager

      - name: ✅ Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": ":white_check_mark: *Deployment Success*\nBranch: `${{ github.ref_name }}`\nTime: `${{ github.event.head_commit.timestamp }}`"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: ❌ Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": ":x: *Deployment Failed*\nBranch: `${{ github.ref_name }}`\nPlease check logs. Rollback has been triggered."
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

   
