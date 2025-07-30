# Celery Task Failures Runbook
## celery config: /etc/systemd/system/celery.service
```
[Unit]
Description=Celery Service
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/Hiringdog-backend

Environment="PATH=/home/ubuntu/shared_venv/bin"
EnvironmentFile=/home/ubuntu/secrets/hiringdog.env
Environment="PYTHONUNBUFFERED=1"

ExecStart=/home/ubuntu/shared_venv/bin/celery -A hiringdogbackend worker --loglevel=info

# Gracefully stop with SIGTERM
KillSignal=SIGTERM
TimeoutStopSec=300

Restart=always
RestartSec=3
TimeoutSec=300
LimitNOFILE=65536

# Log to journal for easier access to logs
StandardOutput=append:/var/log/hiringdog/celery.log
StandardError=inherit

[Install]
WantedBy=multi-user.target
```
## celery-beat:  /etc/systemd/system/celery-beat.service 
```
[Unit]
Description=Celery Beat Service
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/Hiringdog-backend

Environment="PATH=/home/ubuntu/shared_venv/bin"
EnvironmentFile=/home/ubuntu/secrets/hiringdog.env
Environment="PYTHONUNBUFFERED=1"

ExecStart=/home/ubuntu/shared_venv/bin/celery -A hiringdogbackend beat \
  --scheduler django_celery_beat.schedulers.DatabaseScheduler \
  --loglevel=info

Restart=always
RestartSec=3
TimeoutSec=300
LimitNOFILE=65536

StandardOutput=append:/var/log/hiringdog/celery_beat.log
StandardError=inherit

[Install]
WantedBy=multi-user.target
```
### 1. Check Celery Services Status
```bash
# Check Celery worker status
sudo systemctl status celery
sudo systemctl status celery-beat

# Check message brokers
sudo systemctl status rabbitmq-server
sudo systemctl status redis-server

# Check all processes
ps aux | grep celery
ps aux | grep rabbitmq
ps aux | grep redis
```

### 2. Check Celery Logs
```bash
# Celery logs
sudo journalctl -u celery -f
sudo tail -f /var/log/hiringdog/celery.log
sudo journalctl -u celery-beat -f
sudo tail -f /var/log/hiringdog/celery_beat.log

# RabbitMQ logs
sudo journalctl -u rabbitmq-server -f
sudo tail -f /var/log/rabbitmq/rabbit@$(hostname).log

# Redis logs
sudo journalctl -u redis-server -f
sudo tail -f /var/log/redis/redis-server.log

# Check for errors
sudo grep -i "error\|exception\|traceback" /var/log/hiringdog/celery.log | tail -20
sudo grep -i "error\|exception\|traceback" /var/log/hiringdog/celery_beat.log | tail -20
```
### 3. Test Broker Connections

- Test RabbitMQ connection
- Test Redis connection
- Test Celery broker connection

## 🛠Common Failure Scenarios

### Scenario 1: Celery Workers Not Starting

**Symptoms:**
- `systemctl status hiringdog-celery` shows failed
- No Celery processes running
- Tasks not being processed

**Diagnosis:**
```bash
# Check systemd service status
sudo systemctl status celery

# Check service logs
sudo journalctl -u celery -n 50

# Check service configuration
sudo cat /etc/systemd/system/celery.service

```

**Resolution:**
```bash
# Restart Celery service
sudo systemctl restart celery

# If still failing, check configuration
sudo systemctl daemon-reload
sudo systemctl enable celery
sudo systemctl restart celery
```
- Check broker connection
- Check environment variables

### Scenario 2: Celery Beat Runs but Doesn’t Trigger Tasks

**Symptoms:**
celery-beat.service is running, but periodic tasks aren’t firing

**Causes:**
django_celery_beat is not migrated or not registered properly
**Resolution:**
Run migrations:
```bash
python manage.py migrate
```

### Scenario 3: RabbitMQ Connection Issues

**Symptoms:**
- Celery workers disconnected from RabbitMQ
- Tasks stuck in queue
- Connection timeout errors

**Resolution:**
- Restart RabbitMQ service
```
sudo systemctl restart rabbitmq-server
```

- Wait for RabbitMQ to fully start: `sleep 10`
- Restart Celery workers
```
sudo systemctl restart celery
sudo systemctl restart celery-beat
```
- Check RabbitMQ configuration and Confirm CELERY_BROKER_URL: `CELERY_BROKER_URL="amqp://guest:guest@localhost:5672//" `

### Scenario 4: Memory/Resource Issues

**Symptoms:**
- Workers running out of memory
- High CPU usage
- Workers being killed by OOM

**Diagnosis:**
- Check system resources `free -h`, `top`, `htop`
- Too many concurrent workers
- Check Celery worker memory usage
```
ps aux | grep celery | grep -v grep
```

**Resolution:**

- Restart workers to free memory
```
sudo systemctl restart celery
```
- Adjust worker concurrency
`sudo nano /etc/systemd/system/celery.service`: `Add to ExecStart: --concurrency=2`
- Increase memory limits
`sudo nano /etc/systemd/system/celery.service`: `Add: LimitMEMLOCK=infinity`

- Reload and restart
```
sudo systemctl daemon-reload
sudo systemctl restart celery
```

### Quick Recovery
```bash
# Emergency restart all Celery services
sudo systemctl restart celery
sudo systemctl restart celery-beat

# Check if services are running
sudo systemctl status celery celery-beat
```
### Complete Service Reset
```bash
# Stop all Celery services
sudo systemctl stop celery celery-beat
sudo pkill -f celery

# Wait a moment
sleep 10

# Start services in order
sudo systemctl start redis
sleep 5
sudo systemctl start celery
sleep 5
sudo systemctl start celery-beat

# Check status
sudo systemctl status celery celery-beat redis
```


