# 3-Tier-Application-Deployment-on-AWS-EC2


## Architecture at a Glance

```
Browser
  │  HTTP :80
  ▼
┌─────────────────────────┐
│  Frontend EC2           │  Nginx reverse-proxies /api/* → Backend
│  Nginx + React build    │
└──────────┬──────────────┘
           │  Internal :3000
           ▼
┌─────────────────────────┐
│  Backend EC2            │
│  Node.js / Express      │
└──────────┬──────────────┘
           │  Internal :5432
           ▼
┌─────────────────────────┐
│  Database EC2           │
│  PostgreSQL             │
└─────────────────────────┘
```

## Security Group Rules

| Tier       | Port | 
|------------|------|
| Frontend   | 80   | 
| Backend    | 3000 |
| Database   | 5432 |

---

## Step 1 — Frontend EC2

### 1.1 System Packages

```bash
sudo apt update
sudo apt install nginx git -y
```

### 1.2 Clone & Build

```bash
git clone https://github.com/sarowar-alam/single-server-3tier-webapp.git
cd single-server-3tier-webapp/frontend

# Install Node 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y

npm install
npm run build
```

### 1.3 Serve the Build

```bash
sudo rm -rf /var/www/html/*
sudo cp -r dist/* /var/www/html/
```

### 1.4 Nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/default
```

```nginx
server {
    listen 80;
    server_name _;

    root  /var/www/html;
    index index.html;

    # SPA fallback
    location / {
        try_files $uri /index.html;
    }

    # API proxy — strips the /api prefix before forwarding
    location /api/ {
        proxy_pass         http://10.0.15.157:3000/;
        proxy_http_version 1.1;
        proxy_set_header   Host        $host;
        proxy_set_header   X-Real-IP   $remote_addr;
    }
}
```

```bash
sudo nginx -t && sudo systemctl restart nginx
```

---

## Step 2 — Backend EC2

### 2.1 System Packages

```bash
sudo apt update
sudo apt install git nodejs npm -y
```

### 2.2 Clone & Install

```bash
git clone https://github.com/sarowar-alam/single-server-3tier-webapp.git
cd single-server-3tier-webapp/backend
npm install
```

### 2.3 Environment Variables

```bash
nano .env
```

```env
PORT=3000
DATABASE_URL=postgresql://bmi_user:strongpassword@10.0.5.119:5432/bmidb
NODE_ENV=production
FRONTEND_URL=http://13.127.40.217
```

### 2.4 Run with PM2

```bash
sudo npm install -g pm2

pm2 start npm --name "backend-app" -- start
pm2 save
pm2 startup          # follow the printed command to enable on reboot
```

---

## Step 3 — Database EC2

### 3.1 Install PostgreSQL

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

### 3.2 Create Database & User

```bash
sudo -i -u postgres psql
```

```sql
CREATE DATABASE bmidb;
CREATE USER bmi_user WITH ENCRYPTED PASSWORD 'strongpassword';
GRANT ALL PRIVILEGES ON DATABASE bmidb TO bmi_user;
\q
```

### 3.3 Allow Remote Connections

**`postgresql.conf`**

```bash
sudo nano /etc/postgresql/*/main/postgresql.conf
```

```
listen_addresses = '*'
```

**`pg_hba.conf`**

```bash
sudo nano /etc/postgresql/*/main/pg_hba.conf
```

```
# TYPE  DATABASE  USER  ADDRESS      METHOD
host    all       all   0.0.0.0/0    md5
```

```bash
sudo systemctl restart postgresql
```

---

# 📸 Screenshots (Proof of Work)


## 🗄️ Database EC2

* PostgreSQL running
* Database created

📷 Screenshot:

![Alt Text](./screenshots/database-history.png)


---

## 🌐 Application Running

📷 Screenshot:

![Alt Text](./screenshots/frontend-browser.png)
![Alt Text](./screenshots/api-health-from-frontend-ip.png)


---

# 🌍 Application Access Result

Access the application via:

```
http://13.127.40.217/
```

### API Test:

```
http://13.127.40.217/api/health
```

✅ Output:

```json
{
  "status": "ok",
  "environment": "production"
}
```
