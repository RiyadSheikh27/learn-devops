# Next.js VPS Deployment (Ubuntu + Nginx + PM2)

## 1. Install Node.js & Tools (first time only)

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs nginx
sudo npm install -g pm2
sudo systemctl enable nginx && sudo systemctl start nginx
```

---

## 2. Build the App

```bash
cd ~/your-project
npm install
npm run build
```

Check the start port:
```bash
cat package.json | grep -A5 '"scripts"'
```

---

## 3. Start with PM2

```bash
pm2 start npm --name "your-app" -- start
pm2 save
pm2 startup   # copy & run the command it prints
```

---

## 4. Nginx Config

```bash
sudo nano /etc/nginx/sites-available/your-app
```

```nginx
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_IP;

    location / {
        proxy_pass http://localhost:PORT;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/your-app /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 5. Firewall (first time only)

```bash
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
sudo ufw enable
```

---

## 6. Environment Variables

```bash
nano ~/your-project/.env
# paste your vars, save, then:
pm2 restart your-app
```

---

## 7. SSL (after DNS is pointed to VPS)

Add DNS A record: `@ → YOUR_VPS_IP` at your registrar, then:

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

For a subdomain only:
```bash
sudo certbot --nginx -d sub.yourdomain.com
```

---

## Useful Commands

| Task | Command |
|---|---|
| App status | `pm2 status` |
| App logs | `pm2 logs your-app` |
| Restart app | `pm2 restart your-app` |
| Rebuild & restart | `npm run build && pm2 restart your-app` |
| Reload Nginx | `sudo systemctl reload nginx` |
| Nginx error log | `sudo tail -n 50 /var/log/nginx/error.log` |
| Test Nginx config | `sudo nginx -t` |

---

## Common Issues

| Problem | Fix |
|---|---|
| 502 Bad Gateway | App not running or wrong port — check `pm2 status` and Nginx `proxy_pass` port |
| API not working | `.env` missing or wrong values — rebuild after fixing |
| App crashes | `pm2 logs your-app` to see the error |
| SSL fails | DNS not propagated yet — wait 10–30 min and retry |