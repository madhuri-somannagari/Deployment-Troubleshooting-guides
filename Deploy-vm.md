# 🚀 Automated Application Deployment to VM using GitHub Actions (with Rollback Support)

## 📌 Concept Overview

This project automates the deployment of a web application to a virtual machine (VM) using **GitHub Actions**. The solution:

- Delivers seamless application updates from code commits to the live environment.
- Minimizes manual intervention and deployment errors.
- Implements a **rollback mechanism** to revert to the last stable state in case of failure.
- Ensures high availability and faster recovery, especially for production environments.

---

## 🎯 Objective

To implement a **robust and reliable CI/CD pipeline** using:
- **GitHub Actions** for automation,
- **Rsync** for efficient file transfers,
- **Systemd-managed services** like **Gunicorn** and **Celery**,
- **Rollback support** in case of deployment issues,
- Log management with **logrotate**,
- Conditional service restarts based on code changes.

---

## 🧱 Architecture Overview

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
