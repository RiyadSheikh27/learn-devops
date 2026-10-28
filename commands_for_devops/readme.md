# DevOps
### DevOps = Development + Operations
### Life Cycle of DevOps
Plan -> Development -> Build -> Test -> Release -> Deployment -> Monitor
## Linux Commands
#### Basic Commands
```bash
# Current Location/Path
pwd

# All folder and files
ls

# More detailed information
ls -l

# To see all hidden folders/files
ls -a

# To enter any folder
cd <folder_name>

# Create new folder

mkdir <folder_name>

# Create new file 

touch <file_name>

# Enter any file
nano <file_name>

# Delete file
rm -r <file_name>

# Find a file/folder
find -name <file/folder_name>
find Desktop/Riyad -name <file/folder_name>

# Find text inside in file
grep "finding_text" Desktop/file_name.txt
```

#### Shell Scripting
```bash
nano script_file_name.sh
```
```bash
#!/bin/bash
name="Riyad"
echo "Hello, $name"
```

ctrl + s and ctrl + x
```bash
Permission/execulable: chmod +x script_file_name.sh
Run: ./variables.sh
```

## 1. Nginx Site Management

### List all available sites
```bash
ls /etc/nginx/sites-available/
```

### List enabled sites (active)
```bash
ls /etc/nginx/sites-enabled/
```

### Enable a site
```bash
sudo ln -s /etc/nginx/sites-available/site-name /etc/nginx/sites-enabled/
```

### Disable a site
```bash
sudo rm /etc/nginx/sites-enabled/site-name
```

### Reload after changes
```bash
sudo systemctl reload nginx
```

---

## 2. Gunicorn / Backend Services

### List all running services
```bash
systemctl list-units --type=service
```

### Filter Gunicorn services
```bash
systemctl list-units --type=service | grep gunicorn
```

### Check specific service status
```bash
sudo systemctl status service-name
```

### Start service
```bash
sudo systemctl start service-name
```

### Stop service
```bash
sudo systemctl stop service-name
```

### Restart service
```bash
sudo systemctl restart service-name
```

### Enable service at boot
```bash
sudo systemctl enable service-name
```

### Disable service at boot
```bash
sudo systemctl disable service-name
```

---

## 3. Systemd Service Files

### Location of service files
```bash
/etc/systemd/system/
```

### List all service files
```bash
ls /etc/systemd/system/
```

### Edit a service file
```bash
sudo nano /etc/systemd/system/service-name.service
```

### Remove a service file (only if not needed)
```bash
sudo rm /etc/systemd/system/service-name.service
```

### Reload systemd after changes
```bash
sudo systemctl daemon-reload
```

---

## 4. Process Management

### View running processes
```bash
ps aux
```

### Find specific process (Gunicorn/Nginx)
```bash
ps aux | grep gunicorn
ps aux | grep nginx
```

### Check ports in use
```bash
ss -tulnp
```

---

## 5. Safe Cleanup Rules

- Never delete running services without checking status first
- Always run `nginx -t` before reloading Nginx
- Prefer disabling over deleting when unsure
- Use `systemctl stop` before removing services
- Always reload systemd after service changes

---

## 6. Quick Troubleshooting Flow

If something breaks:

1. Check Nginx config:
```bash
sudo nginx -t
```

2. Check service status:
```bash
systemctl status service-name
```

3. Check logs:
```bash
journalctl -u service-name -e
```

4. Reload systemd:
```bash
sudo systemctl daemon-reload
```

---

# End of Guide