# EdisonKart Operations Playbook

This playbook covers local development, production deployment, monitoring, and troubleshooting for the EdisonKart e-commerce platform.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Local Development](#local-development)
3. [Production Deployment](#production-deployment)
4. [Server Management](#server-management)
5. [Database Operations](#database-operations)
6. [Monitoring & Logs](#monitoring--logs)
7. [Troubleshooting](#troubleshooting)
8. [Security Checklist](#security-checklist)
9. [Rollback Procedures](#rollback-procedures)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         PRODUCTION                               │
│                                                                  │
│  ┌──────────┐      ┌──────────┐      ┌──────────────────────┐   │
│  │  Client  │─────▶│  Nginx   │─────▶│  React App (static)  │   │
│  │ Browser  │      │  :443    │      │  /usr/share/nginx/   │   │
│  └──────────┘      └────┬─────┘      └──────────────────────┘   │
│                         │                                        │
│                         │ /api/*                                 │
│                         ▼                                        │
│                   ┌──────────┐      ┌──────────────────────┐    │
│                   │  PM2     │─────▶│  Node.js Backend     │    │
│                   │          │      │  :5000               │    │
│                   └──────────┘      └──────────┬───────────┘    │
│                                                 │                │
│                                                 ▼                │
│                                          ┌───────────┐          │
│                                          │  MongoDB  │          │
│                                          │  Atlas    │          │
│                                          └───────────┘          │
└─────────────────────────────────────────────────────────────────┘

Server: AWS EC2 (13.53.116.48)
Domain: edisonkart.com
```

---

## Local Development

### Prerequisites

- Node.js 18+
- MongoDB (local or Atlas connection)
- Git

### Backend Setup

```bash
cd Edisonkart-backend

# Create environment file
cp .env.example .env

# Edit .env with your values:
# - MONGODB_URI (local: mongodb://127.0.0.1:27017/edisonkart)
# - JWT_SECRET
# - EMAIL_* (Gmail SMTP)
# - RAZORPAY_KEY_* (test keys for dev)
# - OPENAI_API_KEY (optional, for AI features)

# Install dependencies
npm install

# Seed database (creates admin user, sample data)
npm run seed

# Start development server (with hot reload)
npm run dev
```

Backend runs at `http://localhost:5000`.

### Frontend Setup

```bash
cd Edisonkart-frontend

# Create environment file
cp .env.example .env

# Edit .env:
# VITE_API_URL=http://localhost:5000/api
# VITE_GOOGLE_CLIENT_ID=your_google_client_id

# Install dependencies
npm install

# Start development server
npm run dev
```

Frontend runs at `http://localhost:5173`.

### Mobile Setup (Optional)

```bash
cd Edisonkart-mobile

# Edit src/config.js with your API URL
npm install
npx expo start
```

---

## Production Deployment

### Server Requirements

- Ubuntu 22.04+ (recommended)
- Node.js 18+
- Nginx
- PM2 (process manager)
- SSL certificate (Let's Encrypt)

### Initial Server Setup

```bash
# SSH to server
ssh -i your-key.pem ubuntu@13.53.116.48

# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install PM2 globally
sudo npm install -g pm2

# Install Nginx
sudo apt install -y nginx

# Install Certbot for SSL
sudo apt install -y certbot python3-certbot-nginx
```

### Deploy Backend

```bash
# Clone or pull latest code
cd ~/edison-kart-main/Edisonkart-backend

# Create/update .env (NEVER commit this file)
nano .env

# Install dependencies
npm install --production

# Start with PM2
pm2 start src/server.js --name "edisonkart-backend"
pm2 save
pm2 startup  # Follow instructions to enable on boot
```

### Deploy Frontend

```bash
cd ~/edison-kart-main/Edisonkart-frontend

# Create .env
echo "VITE_API_URL=https://edisonkart.com/api" > .env

# Install and build
npm install
npm run build

# Copy to Nginx root
sudo rm -rf /usr/share/nginx/html/*
sudo cp -r dist/* /usr/share/nginx/html/
```

### Nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/edisonkart
```

```nginx
server {
    listen 80;
    server_name edisonkart.com www.edisonkart.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name edisonkart.com www.edisonkart.com;

    ssl_certificate /etc/letsencrypt/live/edisonkart.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/edisonkart.com/privkey.pem;

    root /usr/share/nginx/html;
    index index.html;

    # Frontend (SPA)
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API Proxy
    location /api {
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Authorization $http_authorization;
        
        # Timeouts for long operations
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # File upload size
    client_max_body_size 50M;
}
```

```bash
# Enable site
sudo ln -sf /etc/nginx/sites-available/edisonkart /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default

# Test and reload
sudo nginx -t
sudo systemctl reload nginx

# Setup SSL
sudo certbot --nginx -d edisonkart.com -d www.edisonkart.com
```

### Quick Deploy Script

For subsequent deployments, use:

```bash
#!/bin/bash
# deploy.sh - Run on server

set -e
cd ~/edison-kart-main

# Pull latest
git pull origin main

# Backend
cd Edisonkart-backend
npm install --production
pm2 restart edisonkart-backend

# Frontend
cd ../Edisonkart-frontend
npm install
npm run build
sudo rm -rf /usr/share/nginx/html/*
sudo cp -r dist/* /usr/share/nginx/html/

echo "Deployed at $(date)"
```

---

## Server Management

### PM2 Commands

```bash
# View running processes
pm2 list

# View logs
pm2 logs edisonkart-backend
pm2 logs edisonkart-backend --lines 100

# Restart backend
pm2 restart edisonkart-backend

# Stop backend
pm2 stop edisonkart-backend

# Monitor (CPU, memory)
pm2 monit

# Reload with zero downtime
pm2 reload edisonkart-backend
```

### Nginx Commands

```bash
# Test configuration
sudo nginx -t

# Reload (graceful)
sudo systemctl reload nginx

# Restart
sudo systemctl restart nginx

# View error logs
sudo tail -f /var/log/nginx/error.log

# View access logs
sudo tail -f /var/log/nginx/access.log
```

### SSL Certificate Renewal

Certbot auto-renews, but to manually renew:

```bash
sudo certbot renew --dry-run  # Test
sudo certbot renew            # Actual renewal
```

---

## Database Operations

### MongoDB Atlas

Dashboard: https://cloud.mongodb.com

### Backup

```bash
# Export all collections
mongodump --uri="mongodb+srv://user:pass@cluster.mongodb.net/edisonkart" --out=./backup-$(date +%Y%m%d)

# Export specific collection
mongoexport --uri="..." --collection=products --out=products.json
```

### Restore

```bash
# Restore from backup
mongorestore --uri="mongodb+srv://user:pass@cluster.mongodb.net/edisonkart" ./backup-20240101

# Import collection
mongoimport --uri="..." --collection=products --file=products.json
```

### Useful Queries

```javascript
// Connect via mongosh
mongosh "mongodb+srv://cluster.mongodb.net/edisonkart"

// Count orders
db.orders.countDocuments()

// Recent orders
db.orders.find().sort({createdAt: -1}).limit(10)

// Orders by status
db.orders.aggregate([
  { $group: { _id: "$status", count: { $sum: 1 } } }
])

// Revenue (last 30 days)
db.orders.aggregate([
  { $match: { status: "delivered", createdAt: { $gte: new Date(Date.now() - 30*24*60*60*1000) } } },
  { $group: { _id: null, total: { $sum: "$total" } } }
])
```

---

## Monitoring & Logs

### Application Logs

```bash
# Backend logs (PM2)
pm2 logs edisonkart-backend --lines 200

# Real-time follow
pm2 logs edisonkart-backend -f

# Save logs to file
pm2 logs edisonkart-backend > /tmp/backend-logs.txt
```

### System Monitoring

```bash
# Disk usage
df -h

# Memory usage
free -m

# CPU and processes
htop

# Network connections
netstat -tuln | grep -E '5000|80|443'
```

### Health Check

```bash
# Backend health
curl -s http://localhost:5000/api/products?limit=1 | head -c 100

# Frontend health
curl -s -o /dev/null -w "%{http_code}" https://edisonkart.com

# Full stack check
curl -s https://edisonkart.com/api/categories | jq '.success'
```

---

## Troubleshooting

### Backend Won't Start

```bash
# Check if port is in use
lsof -i :5000

# Check PM2 logs for errors
pm2 logs edisonkart-backend --lines 50

# Verify .env file exists and has required vars
cat ~/edison-kart-main/Edisonkart-backend/.env | grep -E 'MONGODB|JWT|PORT'

# Try running directly to see errors
cd ~/edison-kart-main/Edisonkart-backend
node src/server.js
```

### "Route not found" Errors

- Verify Nginx proxy_pass does NOT have trailing `/api/`
- Check backend is running: `pm2 list`
- Check correct port: `curl http://localhost:5000/api/categories`

### CORS Errors

- Ensure `FRONTEND_URL` in backend `.env` includes the frontend domain
- Multiple origins: `FRONTEND_URL=https://edisonkart.com,http://localhost:5173`

### MongoDB Connection Failed

```bash
# Test connection
mongosh "mongodb+srv://user:pass@cluster.mongodb.net/edisonkart"

# Check if IP is whitelisted in Atlas
# Atlas > Network Access > Add current server IP
```

### SSL Certificate Issues

```bash
# Check certificate expiry
sudo certbot certificates

# Force renewal
sudo certbot renew --force-renewal

# Check Nginx SSL config
sudo nginx -t
```

### Payment Failures

1. Check Razorpay dashboard for error details
2. Verify `RAZORPAY_KEY_ID` and `RAZORPAY_KEY_SECRET` in `.env`
3. Check backend logs: `pm2 logs | grep -i razorpay`

### OpenAI/AI Features Not Working

```bash
# Check if key is set
grep OPENAI_API_KEY ~/edison-kart-main/Edisonkart-backend/.env

# Test key (from server)
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer YOUR_KEY" | head
```

---

## Security Checklist

### Environment

- [ ] `.env` files are NOT in git (check `.gitignore`)
- [ ] Production `.env` has strong `JWT_SECRET`
- [ ] MongoDB user has minimal required permissions
- [ ] API keys are rotated periodically

### Server

- [ ] SSH key-only authentication (disable password)
- [ ] Firewall allows only 22, 80, 443
- [ ] Automatic security updates enabled
- [ ] Fail2ban installed for SSH protection

```bash
# Check firewall
sudo ufw status

# Enable firewall with required ports
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

### Application

- [ ] Rate limiting enabled (already in backend)
- [ ] CORS restricted to known origins
- [ ] Helmet.js headers enabled (already in backend)
- [ ] Input validation on all endpoints
- [ ] No sensitive data in logs

### SSL

- [ ] HTTPS enforced (HTTP redirects to HTTPS)
- [ ] Certificate auto-renewal working
- [ ] Strong SSL configuration (TLS 1.2+)

---

## Rollback Procedures

### Backend Rollback

```bash
# If using git
cd ~/edison-kart-main
git log --oneline -5          # Find last good commit
git checkout <commit-hash>    # Checkout that version
cd Edisonkart-backend
npm install
pm2 restart edisonkart-backend
```

### Frontend Rollback

```bash
# Keep previous builds
cp -r /usr/share/nginx/html /usr/share/nginx/html.backup.$(date +%Y%m%d)

# Rollback
sudo rm -rf /usr/share/nginx/html
sudo mv /usr/share/nginx/html.backup.YYYYMMDD /usr/share/nginx/html
```

### Database Rollback

```bash
# Restore from backup (CAREFUL: overwrites current data)
mongorestore --uri="..." --drop ./backup-YYYYMMDD
```

---

## Contacts & Resources

| Resource | URL |
|----------|-----|
| GitHub Repo | https://github.com/kowshik538/edisonkart |
| MongoDB Atlas | https://cloud.mongodb.com |
| Razorpay Dashboard | https://dashboard.razorpay.com |
| OpenAI API | https://platform.openai.com |
| Server IP | 13.53.116.48 |
| Domain | edisonkart.com |

---

## Changelog

| Date | Change |
|------|--------|
| 2025-01 | Initial playbook created |
