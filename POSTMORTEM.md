# Storage Breaker Deployment and Investigation

## Deployment

The application was deployed following the `README`.

### Prepare application directory

```bash
sudo useradd \
  --system \
  --home /opt/storage-breaker \
  --shell /usr/sbin/nologin \
  appuser

sudo mkdir -p /opt/storage-breaker
sudo mkdir -p /var/log/storage-breaker

sudo chown -R appuser:appuser /opt/storage-breaker
sudo chown -R appuser:appuser /var/log/storage-breaker
```

### Clone application

```bash
sudo -u appuser git clone \
https://github.com/khantnaingset-kns/ape-aws-ec2-assessment-1.git \
/opt/storage-breaker
```

### Install dependencies

```bash
sudo -u appuser -H bash

cd /opt/storage-breaker

python3 -m venv .venv

source .venv/bin/activate

pip install -r requirements.txt
```

### Run application

```bash
/usr/local/bin/storage-breaker app:app \
  --host 127.0.0.1 \
  --port 3000 \
  --workers 1 \
  --no-access-log
```

`/usr/local/bin/storage-breaker` is a symlink to:

```text
/opt/storage-breaker/.venv/bin/uvicorn
```

### Nginx

Configure Nginx to reverse proxy:

```text
http://127.0.0.1:3000
```

---

# Investigation

After approximately 2 hours, the health endpoint returned an unhealthy status.

Investigation found:

* Application continuously wrote `xxx` logs.
* `application.log` grew to **6.3 GB**.
* Root filesystem reached **100%** usage.

```bash
tail -f /var/log/storage-breaker/application.log

df -h

ls -lh /var/log/storage-breaker/application.log
```

---

## Immediate Recovery

To recover the service without manually truncating the log file, configure `logrotate` and rotate the oversized log.

```bash
sudo logrotate -f /etc/logrotate.d/storage-breaker

The application became healthy again after freeing disk space.

---

# Prevention

## Log Rotation

Configure logrotate.

```text
/var/log/storage-breaker/application.log {
    size 500M
    rotate 10
    compress
    delaycompress
    copytruncate
    missingok
    notifempty
    create 0644 appuser appuser
}
```

Create a custom systemd timer to run logrotate every 15 minutes.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now storage-breaker-logrotate.timer
```

Verify:

```bash
systemctl status storage-breaker-logrotate.timer
systemctl status storage-breaker-logrotate.service
```

---

## CloudWatch Agent

Attach the following IAM policy to the EC2 instance:

```text
CloudWatchAgentServerPolicy
```

Install CloudWatch Agent.

```bash
wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb

sudo dpkg -i -E amazon-cloudwatch-agent.deb
```

Configure the agent to:

* Send `/var/log/storage-breaker/application.log` to CloudWatch Logs.
* Publish `disk_used_percent` and `mem_used_percent` metrics.

Start the agent.

```bash
sudo amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
-s
```

Verify:

```bash
systemctl status amazon-cloudwatch-agent
```

---

## CloudWatch Alarms

Create CloudWatch alarms using the metrics published by the CloudWatch Agent.

* `disk_used_percent` > **80%**
* `mem_used_percent` > **90%**

Configure both alarms to send notifications through an SNS topic (Email).

This provides:

* Automatic log rotation to prevent disk exhaustion.
* Centralized log collection in CloudWatch Logs.
* Proactive alerts before disk or memory usage impacts the application.
