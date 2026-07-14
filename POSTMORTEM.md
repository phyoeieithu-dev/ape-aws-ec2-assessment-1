# Storage Breaker Deployment and Investigation

## Deployment

The code has deployed as per instruction in README.

Ran this cmd to have right file ownership on ubuntu user.

```bash
sudo mkdir -p /opt/storage-breaker
sudo chown -R ubuntu:ubuntu /opt/storage-breaker
```

Then ran this command to copy the code that I clone.

```bash
rsync -av ~/ape-aws-ec2-assessment-1/ /opt/storage-breaker/
```

Then I can now activate venv, install dependencies and start the server as instructed with this cmd below.

```bash
cd /opt/storage-breaker

source .venv/bin/activate

pip install -r requirements.txt

.venv/bin/uvicorn app:app \
  --host 127.0.0.1 \
  --port 3000 \
  --workers 1 \
  --no-access-log
```

## Nginx Configuration

Configure Nginx as a reverse proxy from port `80` to:

```
http://127.0.0.1:3000
```

Can test from locally and from nginx too.

---

# Investigation

While checking around 2 hours after running webserver, `curl` request to health check route becomes unhealthy status.

EC2 volume monitoring, average write size had kept increased before server hangs.

I consoled EC2, and checked logs with this cmd and found out application is writing `xxx` logs continously.

```bash
tail -f /var/log/storage-breaker/application.log
```

Also checked `df -h` and root volume has used 100%.

I checked application log file storage usage too.

```bash
ls -lh /var/log/storage-breaker/application.log
```

Output:

```text
-rw-rw-r-- 1 ubuntu ubuntu 6.3G Jul 14 11:26 /var/log/storage-breaker/application.log
```

## Immediate Recovery

For immediate fix to recover, I truncate the log with this cmd and verify log file storage usage.

```bash
sudo truncate -s 0 /var/log/storage-breaker/application.log
```

When curl the health check endpoint, it healthy again.

---

# Prevention

To prevent recurrence this issue, I use `logrotate` on Linux upon log size.

## 1. Create logrotate configuration

Create the logrotate file.

```bash
sudo vi /etc/logrotate.d/storage-breaker
```

Add this configuration.

```text
/var/log/storage-breaker/application.log {
    size 500M
    rotate 10
    compress
    delaycompress
    missingok
    notifempty
    create 0644 ubuntu ubuntu
}
```

---

## 2. Configure a custom systemd timer

A custom systemd timer is configured to run this logrotate.

### Create `/etc/systemd/system/storage-breaker-logrotate.service`

```ini
[Unit]
Description=Run logrotate for storage-breaker

[Service]
Type=oneshot
ExecStart=/usr/sbin/logrotate /etc/logrotate.d/storage-breaker
```

### Create `/etc/systemd/system/storage-breaker-logrotate.timer`

```ini
[Unit]
Description=Run storage-breaker logrotate every 15 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=15min
Unit=storage-breaker-logrotate.service

[Install]
WantedBy=timers.target
```

---

## 3. Reload systemd

```bash
sudo systemctl daemon-reload
```

## 4. Enable the timer

```bash
sudo systemctl enable --now storage-breaker-logrotate.timer
```

## 5. Verify

Check the timer.

```bash
systemctl status storage-breaker-logrotate.timer
```

Verify the service.

```bash
systemctl status storage-breaker-logrotate.service
```

the rotated files will be:

```bash
/var/log/storage-breaker/application.log.1
/var/log/storage-breaker/application.log.2.gz
/var/log/storage-breaker/application.log.3.gz
...
```