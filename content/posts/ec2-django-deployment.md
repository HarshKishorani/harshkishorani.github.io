---
title: "🚀 Complete Django Deployment Guide with EC2"
date: 2025-01-23    
summary: This comprehensive guide will walk you through deploying a Django project on Amazon EC2, complete with WebSocket support HTTPS, and database setup.
categories: 
- Django
- AWS
---

This comprehensive guide will walk you through deploying a Django project on Amazon EC2, complete with WebSocket support HTTPS, and database setup.

![ec2-django-deployment.png](/ec2-django-deployment.png)

## 📚 Key Terms for Beginners

- **EC2**: Amazon Elastic Compute Cloud - A virtual server in the cloud where you can run your applications
- **Django**: A Python web framework for building web applications
- **Gunicorn**: A Python web server that helps run Django applications in production
- **Nginx**: A powerful web server that handles client requests and routes them to your application
- **Supervisor**: A process control system that keeps your application running
- **WebSocket**: A technology that enables two-way communication between client and server
- **Redis**: An in-memory database used for caching and real-time features
- **PostgreSQL**: A powerful, open-source relational database
- **UFW**: Uncomplicated Firewall - A tool to manage server security
- **SSL/HTTPS**: Security protocols that encrypt data between server and client
- **Daphne**: A server for Django Channels, handling WebSocket connections
- **Certbot**: A tool for obtaining free SSL certificates from Let's Encrypt
- **ED25519**: A secure cryptographic key type used for SSH authentication

---

## 🔧 Step-by-Step Deployment Guide

### Step 1: Create EC2 Instance 🖥️
Create an Amazon EC2 instance and save the ED25519 key-value pem file on your computer. This key will be used for secure access to your server.

### Step 2: Configure VS Code Connection 🔌

Add this configuration to your VS Code SSH hosts:
```
Host django-deployment
  HostName <EC2 Public IPv4 DNS>
  User ubuntu
  IdentityFile C:\cert.pem
```

**Alternative Terminal Connection**:
```bash
ssh -i "cert.pem" ubuntu@<EC2 Public IPv4 DNS>
```

**Explanation**: 
- This configuration enables secure SSH access to your EC2 instance
- The `IdentityFile` points to your private key location
- The `User` specifies the default EC2 user (ubuntu)

### Step 3: Install Python and supervisor 📦

```bash
sudo apt install python3-pip python3-dev 
sudo apt install python3-venv
sudo apt install supervisor
```

### Step 4: Project Setup 📂
Clone your Django project and set up the environment:
```bash
# Clone your project to /home/ubuntu/django-project
cd /home/ubuntu/django-project
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Step 5: Calculate Workers 🧮
Get the number of cores your Linux system has using:
```bash
grep -c ^processor /proc/cpuinfo
```

- Gunicorn Worker calculation: (2 × number_of_cores) + 1
- Example: For 2 cores, use 5 workers (2 × 2 + 1)
- This ensures optimal performance without overloading the server

### Step 6: Configure Supervisor 👨‍💼

Create supervisor config file. All supervisor processes go in : `/etc/supervisor/conf.d/`. So, if you ever need to add a new conf file here, you'll just add it there.
```bash
cd /etc/supervisor/conf.d/
sudo vim django-project-service.conf
```

Add this configuration:
```ini
[program:django-project-gunicorn]
user=ubuntu
directory=/home/ubuntu/django-project
command=/home/ubuntu/django-project/venv/bin/gunicorn --workers 1 --bind unix:/home/ubuntu/django-project/app.sock django-project.wsgi:application
autostart=true
autorestart=true
stdout_logfile=/var/log/django-project/gunicorn.log
stderr_logfile=/var/log/django-project/gunicorn.err.log

[program:django-project-daphne]
user=ubuntu
directory=/home/ubuntu/django-project
command=/home/ubuntu/django-project/venv/bin/daphne -b 0.0.0.0 -p 8001 django-project.asgi:application
autostart=true
autorestart=true
stdout_logfile=/var/log/django-project/daphne.log
stderr_logfile=/var/log/django-project/daphne.err.log
```

**Explanation**:
- Configures two processes: Gunicorn (HTTP) and Daphne (WebSocket)
- Sets up automatic process restart
- Establishes log file locations

Activate supervisor processes:
```bash
supervisorctl reread
supervisorctl update
```

- `reread` loads the new configuration
- `update` applies the changes

Check status:
```bash
sudo supervisorctl status
```

### Step 7: Configure Nginx 🌐

Install Nginx and UFW:
```bash
sudo apt-get install nginx ufw -y
```

Configure firewall: We want ssh and Nginx to work. **Nginx Full** allows for both **http** (80) and **https** (443) connections. 
```bash
sudo ufw allow ssh
sudo ufw allow 'Nginx Full'
sudo ufw enable
```
If you're not kicked off your ssh session, you can now proceed.

Verify firewall status:
```bash
sudo ufw status numbered
```

Expected output:
```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere                  
[ 2] Nginx Full                 ALLOW IN    Anywhere                  
[ 3] 22/tcp (v6)               ALLOW IN    Anywhere (v6)             
[ 4] Nginx Full (v6)           ALLOW IN    Anywhere (v6)  
```

Remove default Nginx site:
```bash
rm /etc/nginx/sites-enabled/default
```

Get local IP:
```bash
localip=$(hostname -I | cut -f1 -d' ')
echo $localip
```

Create Nginx configuration:
```bash
sudo vim /etc/nginx/sites-available/django-project.conf
```

Add this configuration:
```nginx
server {
    server_name localip;
    listen 80;
    listen [::]:80;

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/ubuntu/django-project/app.sock:;
        proxy_buffer_size       128k;
        proxy_buffers           4 256k;
        proxy_read_timeout      60s;
        proxy_busy_buffers_size 256k;
        client_max_body_size    2M;
    }

    location /ws/ {
        proxy_pass http://127.0.0.1:8001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout  86400;
    }
}
```

Link configuration: Link the **sites-available** with sites-enabled in nginx (Changes in sites-available reflect in sites-enabled) : 
```bash
ln -s /etc/nginx/sites-available/django-project.conf /etc/nginx/sites-enabled/django-project.conf
```

Reload Daemon and Nginx:
```bash
systemctl daemon-reload
sudo systemctl reload nginx
```

### Step 8: SSL Configuration 🔒

Get a free subdomain (Or you can use you own domain as well. Refer [this](https://www.codingforentrepreneurs.com/blog/custom-domain-and-https-with-lets-encrypt))
1. Visit https://freedns.afraid.org/subdomain and get yourself a free subdomain.
2. Add your EC2's public IP address

Install Certbot:
```bash
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update -y
sudo apt-get install python-certbot-nginx -y
```

Update Nginx configuration with domain:
```bash
sudo vim /etc/nginx/sites-available/django-project.conf
```

Update server name from public ip of ec2 to this domain name:
```nginx
server {
    server_name your-domain.freedns.org;
    # ... rest of the configuration ...
}
```

Verify configuration in linked sites-enabled file:
```bash
cat /etc/nginx/sites-enabled/django-project.conf
```

Get SSL certificate:
```bash
sudo certbot --nginx -d your-domain.freedns.org
```

Verify auto-renewal:
```bash
sudo certbot renew --dry-run
```

### Step 9: Redis Setup 📡

Install and configure Redis:
```bash
sudo apt update
sudo apt install redis-server -y
```

Enable and start Redis:
```bash
sudo systemctl enable redis-server.service 
sudo systemctl start redis 
sudo systemctl status redis
```

### Step 10: PostgreSQL Setup 🗃️

Install PostgreSQL:
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

Verify installation:
```bash
psql --version
```

Ensure PostgreSQL is running and set to start on boot:
```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo systemctl status postgresql
```

Switch to **postgres** user and open PostreSQL shell:
```bash
sudo -i -u postgres
psql
```

In PostgreSQL shell: To Update the **postgres** user's password and to quit the shell.
```sql
\password postgres
\q
```

Exit PostgreSQL user:
```bash
exit
```

Update Django settings:
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'postgres',
        'USER': 'postgres',
        'PASSWORD': 'your_password',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```

Apply migrations:
```bash
source /home/ubuntu/django-project/venv/bin/activate
python manage.py migrate
```

And you're live.

---

## 📝 Log File Locations

### Nginx Logs 📊
```bash
# Error Log
sudo tail -n 50 /var/log/nginx/error.log

# Access Log
sudo tail -n 50 /var/log/nginx/access.log
```

### Supervisor Logs 📈
```bash
# Main Log
sudo tail -n 50 /var/log/supervisor/supervisord.log

# Gunicorn Logs
sudo tail -n 50 /var/log/django-project/gunicorn.log
sudo tail -n 50 /var/log/django-project/gunicorn.err.log

# Daphne Logs
sudo tail -n 50 /var/log/django-project/daphne.log
sudo tail -n 50 /var/log/django-project/daphne.err.log
```

---

## 🔄 When to Restart What

| Component      | Restart Needed?  | Reason                       |
| -------------- | ---------------- | ---------------------------- |
| Gunicorn       | ✅ Always         | Required for code changes    |
| Daphne         | ✅ For WebSockets | Needed for WebSocket changes |
| Nginx          | ❌ Usually not    | Only for config changes      |
| Redis/Postgres | ❌ Rarely         | Only for DB/cache changes    |

## 🛠️ Restart Commands

After code updates:
```bash
# Activate virtual environment
source /home/ubuntu/django-project/venv/bin/activate

# Apply changes
python manage.py migrate
python manage.py collectstatic --noinput

# Restart services
systemctl daemon-reload
sudo systemctl reload nginx

# Restart supervisor processes
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart all

# Or restart specific services
sudo supervisorctl restart django-project-gunicorn
sudo supervisorctl restart django-project-daphne
```

## 🎯 TODO List
- [ ] Add Huey Tasks using supervisor
- [ ] Implement automatic Nginx refresh after code push

## 📚 References
- [Supervisor and Gunicorn Setup](https://www.codingforentrepreneurs.com/blog/hello-linux-setup-gunicorn-and-supervisor)
- [Nginx and UFW Configuration](https://www.codingforentrepreneurs.com/blog/hello-linux-nginx-and-ufw-firewall)
- [Nginx 502 Bad Gateway Fix](https://stackoverflow.com/a/70120803)
- [Daphne Deployment Guide](https://channels.readthedocs.io/en/latest/deploying.html)
- [Custom Domain and HTTPS Setup](https://www.codingforentrepreneurs.com/blog/custom-domain-and-https-with-lets-encrypt)