## Automated Application Deployment to VM using GitHub Actions (with Rollback Support)

## Concept Overview

This project automates the deployment of a web application to a virtual machine (VM) using **GitHub Actions**. The solution:

- Delivers seamless application updates from code commits to the live environment.
- Minimizes manual intervention and deployment errors.
- Implements a **rollback mechanism** to revert to the last stable state in case of failure.
- Ensures high availability and faster recovery, especially for production environments.

---

## Objective

To implement a **robust and reliable CI/CD pipeline** using:
- **GitHub Actions** for automation,
- **Rsync** for efficient file transfers,
- **Systemd-managed services** like **Gunicorn** and **Celery**,
- **Rollback support** in case of deployment issues,
- Log management with **logrotate**,
- Conditional service restarts based on code changes.

---

## Architecture Overview

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
## Deployment Workflow
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
## Deployment Strategy 
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
   
