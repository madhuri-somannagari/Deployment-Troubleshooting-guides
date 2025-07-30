## Load Testing & Auto-Scaling Strategy

To conduct effective load testing and prepare for auto-scaling, we use **Locust**, a Python-based load testing tool. This process helps identify system bottlenecks, max user capacity, and prepares your infrastructure to scale reliably under load.

### Objective

- Evaluate the system’s performance under concurrent and sustained load using Locust
- Identify the maximum concurrent users the current infrastructure can support
- Plan an efficient vertical and horizontal scaling strategy

### Tool: Locust

**Locust** is an open-source Python-based load testing tool for simulating thousands of concurrent users. It allows flexible scripting of user behavior and generates meaningful performance metrics.

### Test Scenarios

#### 🔸 High Burst Load: `500 Interviews/Minute`
- ≈ 8.3 requests/second
- Simulates many users scheduling interviews concurrently

#### 🔸 Sustained Load: `2000–4000 Interviews/Day`
- ≈ 83–167 per hour (1.4–2.8 per minute)
- Simulates real-time traffic across the day

### Preparing the Environment

- Always use a **test/staging** environment — _never run load tests on production without explicit approval_.
- Set up your backend application locally or on a staging VM.
- Clone your application repo and create a virtual environment.
- Install Locust:
```bash
pip install locust
```
#### Write Locust Test Script
Create a file called locustfile.py:
```
from locust import HttpUser, task, between, SequentialTaskSet
# ... (rest of your code) ...

class StagingUser(HttpUser):
    wait_time = between(1, 3)
    host = "http://localhost:8000"
    @task
    def login(self):
        self.client.post("/api/login/", json={
            "email": "madhuri@hiringdog.com",
            "password": "Madhu@9182"
            })
```
Adjust the endpoint and payload to reflect your application behavior accurately.

#### Running the load tests
### Step 1: Run Locust
From the directory with your locustfile.py
``` bash
Locust -f locustfile.py
```
### Step 2: Access Locust Web UI
Open http://localhost:8089 in your browser. Fill in:
- Number of users (e.g., 500)
- Spawn rate (e.g., 8–10 users/second)
- Target host (e.g., http://localhost:8000 or your deployed URL)
### Step 3: Monitor System Performance
##### Use the Locust Web UI to track:
- Response times (median, 95th percentile)
- Throughput (requests/sec)
- Error rates (4xx, 5xx)
##### Use server-side tools to monitor:
- CPU/Memory usage: top, htop, or GCP Monitoring
- Database load: connection limits, query performance
#### Key Metrics to Measure
- Response Time (especially 95th percentile)
- Requests/Second (Throughput)
- Error Rate (timeouts, HTTP 500s)
- System Resources: CPU, Memory, DB connections
#### Load Testing Strategy
Initial Test
- Start with 50–100 users
- Gradually ramp up until degradation is noticed
- Record system behavior and bottlenecks
Heavy Load Example: Simulating 500 Schedules/Minute:
- Set Locust to 500 users
- Spawn rate: 8–10 users/second
Watch for:
- HTTP 5xx errors (e.g., 500, 502, 504)
- Long response times (>2–3 seconds)
- High resource usage

#### Preparing for Auto-Scaling
#### If you hit resource limits, consider:
- Increasing VM size (vertical scaling)
- Adding more VMs (horizontal scaling, especially for Celery workers)
- Setting up GCP Managed Instance Groups for web servers.
#### Start low and ramp up:
- Begin with 50–100 users
- Start Locust, set the number of users and spawn rate to match your scenario.
- Observe how your system performs (response times, errors, resource usage).
- Gradually increase up to 1000+ users or until degradation
#### Analyze Results
- How many requests succeed/fail?
- What is the average response time?
- Does CPU/RAM spike on your VMs?
- At what point does the system slow down or fail?
- Auto-Scaling Preparation: Based on test results:
##### Vertical Scaling
- Upgrade VM instance types (more CPU/RAM)
##### Horizontal Scaling
- Add more VM instances
- Use GCP Managed Instance Groups for auto-scaling web servers
- Consider scaling Celery workers if job queuing is affected
### Final Notes
- Always test in an isolated environment
- Never run high load on production unless planned
- Use database connection pooling and Django settings like CONN_MAX_AGE = 60
- Monitor your infrastructure continuously using GCP Monitoring, logs, and DB metrics

