
## Django Application Failures Runbook

### check Gunicorn configuration
file path: ` /etc/systemd/system/gunicorn.service `
```
[Unit]
Description=Gunicorn instance to serve Hiringdog
After=network.target

[Service]
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/Hiringdog-backend

Environment="PATH=/home/ubuntu/shared_venv/bin"
EnvironmentFile=/home/ubuntu/secrets/hiringdog.env
Environment="PYTHONUNBUFFERED=1"

ExecStart=/home/ubuntu/shared_venv/bin/gunicorn \
  --workers 2 \
  --pid /run/gunicorn/gunicorn.pid \
  --bind unix:/run/gunicorn/gunicorn.sock \
  hiringdogbackend.wsgi:application

ExecReload=/bin/kill -s USR2 $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID

PIDFile=/run/gunicorn/gunicorn.pid
RuntimeDirectory=gunicorn
RuntimeDirectoryMode=0755
Restart=always

# Redirecting standard output and error to log files
StandardOutput=append:/var/log/hiringdog/gunicorn.log
StandardError=append:/var/log/hiringdog/gunicorn_error.log

# Set limits
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
```

### 1. Check Application Status
```bash
# Check if Django process is running
sudo systemctl status gunicorn
ps aux | grep manage.py

# Check port availability
netstat -tlnp | grep :8000
lsof -i :8000

# Test application response
curl -I http://localhost:8000/
curl -I https://prod-api.hdiplatform.in/ 
curl -I https://api.hdiplatform.in/
```

### 2. Check Logs
```bash
# Gunicorn application logs
sudo journalctl -u gunicorn -f
sudo tail -f /var/log/hiringdog/gunicorn_error.log
sudo tail -f /var/log/hiringdog/gunicorn.log

# Error logs
sudo tail -f /var/log/hiringdog/app_error.log
sudo tail -f /var/log/hiringdog/errors.log
```

## 🛠️ Common Failure Scenarios

### Scenario 1: Gunicorn Process Not Starting

**Symptoms:**
- `sudo systemctl status gunicorn` shows failed
- Port 8000 not listening
- Application not responding

**Diagnosis:**
```bash
# Check systemd service status
sudo systemctl status gunicorn --no-pager

# Check service logs
sudo journalctl -u gunicorn --no-pager -n 50
```

**Resolution:**
```bash
# Restart the service
sudo systemctl restart gunicorn

# If still failing, check configuration
sudo systemctl daemon-reload
sudo systemctl enable gunicorn
sudo systemctl restart gunicorn

# Check for missing dependencies
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
pip install -r requirements.txt
```

### Scenario 2: Memory/Resource Exhaustion

**Symptoms:**
- Slow response times
- Out of memory errors in logs
- Process killed by OOM killer

**Diagnosis:**

Check system resources
`free -h`, `df -h`, `top`, `htop`

Check Django process memory usage
``` bash
ps aux | grep gunicorn
ps aux --sort=-%mem | head -10
```

**Resolution:**

Restart application to free memory
```bash
sudo systemctl restart gunicorn
```
Reduce Gunicorn workers: `gunicorn --workers=2`

If persistent, increase memory limits
Edit systemd service file if mandatory: `sudo nano /etc/systemd/system/gunicorn.service `: `Add: LimitMEMLOCK=infinity`

Reload and restart
```bash
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
```

### Scenario 3: Import/Module Errors

**Symptoms:**
- `ImportError`, `SyntaxError` in logs
- ModuleNotFoundError
- Application fails to start

**Resolution:**
```bash
# Reinstall dependencies
cd /home/ubuntu/Hiringdog-backend
source venv/bin/activate
pip install -r requirements.txt --force-reinstall

# check application code
Fix any syntax, import, or environment variable issues.

# Update Django settings
python manage.py check
```
### scenario 4: Invalid File Permissions or Ownership

**symptoms:**
- Permission denied: '/run/gunicorn.sock'
- Failed to open log file: Permission denied

**Resolution:**
Fix service file permissions:
```
sudo chown root:root /etc/systemd/system/gunicorn.service
sudo chmod 644 /etc/systemd/system/gunicorn.service
```
Adjust ownership and permissions for socket and log files:
```
sudo chown ubuntu:ubuntu /run/gunicorn.sock
sudo chmod 660 /run/gunicorn.sock
```

### Debug Mode Investigation
```
# Enable debug logging temporarily
# Edit settings.py: DEBUG = True
# Add to LOGGING configuration

# Check for specific errors
grep -r "Exception" /var/log/hiringdog/
grep -r "Traceback" /var/log/hiringdog/
```

## 🔗 Related Runbooks

- [Database Failures](./database-failures.md)
- [Celery Failures](./celery-failures.md)
